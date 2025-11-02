# TFTSS
'TFTSS' is an automated unit test generation tool that combines a requirements-driven assertion checking mechanism with an iterative high-coverage test generation strategy.'TFTSS' decomposes assertions into "test expressions" and "actual values," utilizing system functional requirements to guide LLM(gpt-3.5-turbo and gpt-4o) to generate more accurate assertions.TFTSS analyzes the initial test cases, extracts uncovered code lines, generates semantically independent code snippets, and provides targeted hints for LLM to generate additional tests. Mutation testing is used to further optimize the quality and effectiveness of the tests.
![](https://github.com/ZIYUANLI-star/TFTSS/blob/main/TFTSS.png)
## 1. Module Under Test Detector
The module-under-test detector is responsible for analyzing the structure of source code to extract functions,methods,classes, and other elements, as well as identifying their interdependencies and required packages. 

## 2. Test Case Generator
TFTSS employs a large language model (LLM) to auto matically generate initial test cases for the modules under test. 

## 3.  Syntax Corrector
We first use Python’s AST module to parse the unit test suite and generate an abstract syntax tree (AST) to ensure compliance with Python’s syntax rules.If the parsing process identifies a syntax error, we create a prompt marker based on the error type (for example, "Please correct the syntax error in the following code:") and rely on LLM’s syntax repair function for preliminary repairs. If syntax errors still exist, the runtime error corrector in the execution phase will make additional repairs to ensure that the test case can eventually be successfully executed.

## 4. Runtime Error Corrector
TFTSS monitors and records any runtime errors after executing the test suite. It then con structs prompt tokens from the captured error information and invokes the LLMtoperform targeted fixes. Once repairs are completed, the test suite is executed again, and this process is iterated as needed. Experiments show that this “detect–repair–retest” loop effectively resolves the majority of runtime errors, substantially improving the executability and stability of the test suite.

## 5.  Assertion Actual-Value Checker
TFTSS first extracts the main expression of each assertion (i.e., the left side of the assertion) and combines it with the functional annotations derived from the tested module into a hint label, while referencing the type information of the originally generated assertion value. Guided by these contextual elements, the LLMsimulates the reasoning of professional test engineers and regenerates appropriate "actual values" consistent with the functional description. This way, the updated assertions retain the original type requirements while accurately reflecting the expected functionality of the code under test,resulting in a more targeted and higher-quality unit test suite.

## 6. Test Case Augmenter
TFTSS first leverages Python’s Coverage tool to analyze the test suite coverage to identify specific lines in the tested module that are still not covered. Given that generating test cases at the granularity of smaller code snippets can help LLM achieve highercoverage,wecombinetheseuncoveredlineswith their surrounding context to form semantically complete uncovered code snippets. Subsequently, the function code containing these uncovered snippets is provided as a hint prefix, and hint markers are made to highlight that covering these specific snippets is a core requirement. In this process,we strategically reduce the total amount of context provided to LLM and instead increase the level of detail of certain key elements. This step-by-step approach can better guide LLM to understand the functionality and intent behind the

## 7. Test Failure Triage Module
 the unit test suite that triggers execution failures is taken as input. A three-source fusion strategy for test failure triage combines static analysis, LLM analysis, and dynamic analysis. The failing tests are thereby partitioned into three distinct collections: code defects test-case set, test defects test-case set, and an ambiguous test-case set. 

## 8. Feedback Module
TFTSS first executes the test suite to identify and isolate all test cases where the expected and actual values do not match. It then uses these faulty test cases along with the code under test to formulate appropriate hints. By leveraging such test cases, TFTSS uses a large language model (LLM) to perform feedback-based inspection of the code under test and generate modification suggestions. This last step completes the full cycle of unit testing.

## Example
Here is an example of `PUT` from the `apimd` dataset. 
```python
#the PUT92
class TimerError(Exception):
    pass

class Timers:
    def add(self, name: str, duration: float):
        pass

FloatArg = Union[str, float]

@dataclass
class Timer(ContextDecorator):
    """Time your code using a class, context manager, or decorator."""

    timers: ClassVar[Timers] = Timers()
    name: Optional[str] = None
    text: Union[str, Callable[[float], str]] = "Elapsed time: {:0.4f} seconds"
    initial_text: Union[bool, str] = False
    logger: Optional[Callable[[str], None]] = print
    last: float = field(default=math.nan, init=False, repr=False)
    _start_time: Optional[float] = field(default=None, init=False, repr=False)

    def start(self) -> None:
        """Start a new timer."""
        if self._start_time is not None:
            raise TimerError("Timer is running. Use .stop() to stop it")

        # Log initial text when timer starts
        if self.logger and self.initial_text:
            if isinstance(self.initial_text, str):
                initial_text = self.initial_text.format(name=self.name)
            elif self.name:
                initial_text = f"Timer {self.name} started"
            else:
                initial_text = "Timer started"
            self.logger(initial_text)

        self._start_time = time.perf_counter()

    def stop(self) -> float:
        """Stop the timer, and report the elapsed time."""
        if self._start_time is None:
            raise TimerError("Timer is not running. Use .start() to start it")

        # Calculate elapsed time
        self.last = time.perf_counter() - self._start_time
        self._start_time = None

        # Report elapsed time
        if self.logger:
            if callable(self.text):
                text = self.text(self.last)
            else:
                attributes = {
                    "name": self.name,
                    "milliseconds": self.last * 1000,
                    "seconds": self.last,
                    "minutes": self.last / 60,
                }
                text = self.text.format(self.last, **attributes)
            self.logger(str(text))
        if self.name:
            self.timers.add(self.name, self.last)

        return self.last

    def __enter__(self) -> "Timer":
        """Start a new timer as a context manager."""
        self.start()
        return self

    def __exit__(self, *exc_info: Any) -> None:
        """Stop the context manager timer."""
        self.stop()

```

You can run the `run_test.py` file in TFTSS.

The final test case that `TFTSS` generates for this example `PUT` is as follows:
```python
def test():
   
    timer = Timer()
    timer.start()
    assert timer._start_time is not None, 'Timer failed to start'

    time.sleep(1)
    elapsed_time = timer.stop()
    assert math.isclose(elapsed_time, 1, rel_tol=0.05), 'Timer failed to measure time correctly'

    with Timer() as t:
        time.sleep(1)
    assert math.isclose(t.last, 1, rel_tol=0.05), 'Timer failed to measure time correctly as a context manager'

    try:
        timer.start()
        timer.start()
    except TimerError:
        pass
    else:
        assert False, 'TimerError not raised when starting a running timer'

    try:
        timer = Timer()
        timer.stop()
    except TimerError:
        pass
    else:
        assert False, "TimerError not raised when stopping a timer that hasn't started"

    timer.start()
    time.sleep(0.1)
    elapsed_time = timer.stop()
    assert elapsed_time >= 0.1

    with Timer() as timer_cm:
        time.sleep(0.1)
    assert timer_cm.last > 0

    with patch('time.perf_counter', return_value=124.456):
        timer.start()
        timer._start_time = 123.456
        elapsed_time = timer.stop()
    assert elapsed_time == 1.0

    with patch('time.perf_counter', side_effect=[200.0, 201.0]):
        with Timer() as cm_timer:
            pass
    assert cm_timer.last == 1.0

    # Additional test using MagicMock
    self_timer = MagicMock()
    self_timer.__enter__.return_value = self_timer
    self_timer.__exit__ = MagicMock()
    result = self_timer.__enter__()
    assert self_timer.start.called, 'The start method should be called when entering the context.'

if __name__ == "__main__":
    test()

  ```
