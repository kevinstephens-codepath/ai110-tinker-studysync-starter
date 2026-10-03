# TF Notes - 1B
## Part 1
Manually added a test for (59) asserting that it returns "Skip".  
Ran pytest and confirmed test is passing

I prompted Claude for some test cases as well:
> **Prompt**: Without editing files, please explain a few test cases that would help find boundary values for the session_rating function  
>
> **Answer**: Here are boundary-focused test cases for session_rating() (scoring.py:14-24), which buckets an int into 5 grade strings via cascading >= checks:
>
>Boundary values (each threshold edge, since off-by-one errors hide at boundaries):
>
>session_rating(90) → "Great" and session_rating(89) → "Good" — confirms the 90 cutoff is inclusive on the right side.
>session_rating(80) → "Good" and session_rating(79) → "OK"
>session_rating(70) → "OK" and session_rating(69) → "Meh"
>session_rating(60) → "Meh" and session_rating(59) → "Skip"
>Each pair should be its own test (or parametrized) so a failure points at exactly which boundary shifted.
>
>Extreme/degenerate values:
>
>session_rating(0) → "Skip" — lower bound of the "expected" range.
>session_rating(100) → "Great" — natural max if minutes+focus+bonus caps at 100.
>Breaker inputs (the ones already stubbed out in run_demo() at scoring.py:48-53) — these test whether the function should validate input at all:
>
>session_rating(-5) → currently falls through to "Skip" silently. Is a negative score even valid input? If minutes/focus/streak bonus can never go negative upstream, this may not need guarding — but worth deciding explicitly.
>session_rating(101) or higher → currently returns "Great" since there's no upper bound check. Fine if scores are unbounded-but-always-great above 90, but worth confirming that's intentional.
>session_rating(87.5) (float) → works today because >= compares fine with floats, returns "Good". Not actually a breaker, but worth a test to lock in that non-int numeric input is tolerated (the type hint says int but Python won't enforce it).
>session_rating("80") (str) → raises TypeError on the first >= comparison. This is where the "breaker" label actually applies — decide if you want a clean TypeError/ValueError or to let it fail naturally.
>session_rating(None) → also raises TypeError for the same reason.
>The real design decision from the ticket is: should session_rating validate/clamp input (reject str/None, clamp negatives or >100), or is it fine to trust callers since apply_streak_bonus and the sliders already constrain the range? That determines whether the str/None/negative cases become pytest.raises(TypeError) assertions or trigger new validation code.

> Q: Why test a function that already works?  
> A: Because you want to assert that it works at runtime and set a baseline test case to see where your code was at

## Part 2
Refactored apply_streak_bonus to scoring_helpers.py  
Imported apply_streak_bonus from scoring_helpers.py to scoring.py  
Tested both scoring.py and test_scoring.py, both ran and passed, no issues  

> Q: What would break if the import in scoring.py pointed to the wrong file?  
> A: Pylance would give a warning that the import is missing, and when run, scoring and test_scoring would error/ not pass assertion tests due to relying on apply_streak_bonus not being present.

## Part 3
Added session_rating calls with breaker inputs such as int over 100, negative int, float, str, and None to run_demo()  
Ran scoring.py. Here are the results of the breaker runs:
- int over 100: Great
- negative int: Skip
- float: OK
- str: Error (evaluating int vs str)
- None: Error (evaluating int vs None)

Decisions:
- int over 100, this should indicate that something is wrong with our ranking system and should produce an error
- negative int, this should also indicate that something is wrong with our ranking system and should produce an error
- float, this is ok, if we miss a conversion to int somewhere, it's reasonable to let the scoring work
- str, this type should never be passed to session_rating, therefore it should produce an error
- None, this type should never be passed to session_rating, therefore it should produce an error

> Q: Which breaker input surprised you most, and why?  
> A: Honestly I looked at session_rating beforehand so none of the results suprised me.

## Part 4

> Q: "state, in one sentence: what moved, one AI suggestion you verified, one breaker input that mattered."  
> A: The function that moved was apply_streak_bonus, I verified the one AI explanation I recieved, and the breaker that mattered would have to be str, as it would be significant for the program to pass a string to session_rating without human intervention somewhere.

# TF Notes - 2B

## Part 1
Tested the bug, and asked claude for why it doesn't correctly update.

> A. The cause is in sessions.py:57-61. Streamlit re-executes the whole script from top to bottom on every interaction, and a plain local variable doesn't survive that.
> count = 0
> if st.button("Log a session (broken)"):
>     count += 1
> st.metric("Sessions logged (broken)", count)
> Here is what happens when you click the button:
> Click. Streamlit starts a new run of the script. This is a rerun, not a callback into the existing run.
> count = 0 runs first. The new run has a fresh count and nothing carries over from the previous run. The old value is gone.
> st.button returns True for this one run only. That makes count equal to 1, and st.metric briefly shows 1.
> The next interaction reruns the script. Any click, including one on the "fixed" button or a widget elsewhere, resets count to 0, and the button returns False. The metric goes back to 0.
> So the counter never gets past 1, and that 1 is visible only on the run triggered by the click itself. Button state works the same way. A button is True for a single rerun and False afterwards, so it can't hold a running total.
> The fix is to keep the count in st.session_state. That is the per-session store that persists across reruns:
> if "fixed_count" not in st.session_state:
>     st.session_state.fixed_count = 0   # initialize once, not on every rerun
> if st.button("Log a session (fixed)"):
>    st.session_state.fixed_count += 1
> st.metric("Sessions logged (fixed)", st.session_state.fixed_count)
> The if "fixed_count" not in ... guard matters. If you wrote st.session_state.fixed_count = 0 unconditionally, it would reset the count on every rerun, which is the same bug you started with.
> The TODO at sessions.py:63-67 is where this change goes. I haven't edited anything. I can apply it if you want.  

Manually added initializing the session_state fixed_count and fixed the label/metric to display the correct value. Reran app, and fixed button updates correctly.  

>Q. In your own words, what does "rerun" mean here?  
>A. Every time you click something in a streamlit app, streamlit runs the entire python script again from the first line, to the last. Picture a whiteboard that gets wiped clean and redrawn from scratch every time anyone touches the board. Your python script is the recipe for rewritting that whiteboard and after every click on a streamlit app, that recipe is rerun/rewritten to the board, anything you did outside the recipe is wiped and only comes back if you manually add it back.

## Part 2
Tested that add session adds even with empty input.  
Implemented validation for both whitespace and duration.  
Reran and tested that validation works correctly.  

>Q. What's one input a user might type that you haven't tested yet?  
>A. Something I did not test for is emojis, Tested after this question and they work in the subject line, and you can't type them in the duration, it only allows whole numbers.

## Part 3
Ran sessions.py standalone.  
Wrote the SesssionDC class as a dataclass.  
Implemented SessionDC as an object and printed it same as the original class.  
Both print the same thing except for the class name.  

>Q. What did @dataclass generate that you didn't write yourself?  
>A. The @dataclass genereates __init__ and __repr__ methods without having to write them. Allows you to have a simple class with just the variables and methods you need without having to write a specific __init__ or __repr__ method.

## Part 4
Implemented both find_conflicts(sessions) and next_occurrance(last_date, frequency).  
Tested running sessions.py and everything returns as expected, no errors either.

>Q. Why does find_conflicts need to handle an empty list without crashing?  
>A. find_conflicts needs to handle empty lists due to it being an expected value that should be allowed to happen, if there are no conflicts, or there are no values to check against yet.