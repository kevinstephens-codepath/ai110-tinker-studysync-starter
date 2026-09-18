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