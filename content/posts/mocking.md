---
date: '2025-01-05'
title: 'How mocking works'
draft: false
tags: ["python"]
comments: true
showToc: false
TocOpen: false
hidemeta: false
# description: "Desc Text."
# canonicalURL: "https://canonical.url/to/page"
disableHLJS: true # to disable highlightjs
disableShare: true
disableHLJS: false
hideSummary: true
searchHidden: false
ShowReadingTime: true
ShowBreadCrumbs: false
ShowPostNavLinks: false
ShowWordCount: false
ShowRssButtonInSectionTermList: false
UseHugoToc: false
---
One of the issues I struggled with at the beginning of my engineering journey was understanding how mocking works. For some reason I had a hard time wrapping my head around it, and when things finally clicked, I had an incredibly satisfying AHA moment. This article is aimed at developers who are curious about mocking, as well as at my younger self. I wish I found this article back when I was still strugling with this concept.

### What's mocking

Mocking is used in unit testing to replace real objects with mock objects (mocks). Mocks mimic the behavior of real objects and allow developers to test their code without relying on external dependencies. Dependencies such as API calls, database calls, environment variables, datetime objects, file-like objects, and user input can be mocked. By having full control over their behavior, developers can use mocks to deterministically simulate and test specific scenarios.

### Setting things up

To demonstrate how mocking works, let's write a function that makes an API call to [GitHub's users API](https://api.github.com/users) and a couple of unit tests to test it. If you want to follow along, you can configure a virtual environment.

```shell
python -m venv venv
source venv/bin/activate
pip install requests
```

Then create a file called `get_github_user.py` and paste the below code.


```python
import requests

def get_github_user(user: str) -> dict:
    response = requests.get(f"https://api.github.com/users/{user}")
    if response.status_code != requests.codes.ok:
        raise Exception(response.reason)
    return response.json()
```

To properly test our function we'll need 2 unit tests. The first test covers the happy path scenario, where the API request returns a 200 response and a dictionary. The second test covers the exception scenario, where the API request returns a non 200 response and a reason. The `requests.get(f"https://api.github.com/users/{user}")` API call is our dependency. If we don't mock the dependency, the request will be made on every test run. Not mocking API calls in unit tests is bad for a multitude of reasons:
* GitHub's API has to behave as expected and not change.
* Network issues can cause the unit tests to fail.
* Test execution time can significantly increase.
* Non-free API requests can substantially increase cost.
* Higher bandwidth and resource consumption.
* Hard to test multiple scenarios.

### Let's mock

Let's write our happy path unit test and mock the API call. I used [pytest](https://docs.pytest.org/en/stable/) for testing and [pytest-mock](https://pytest-mock.readthedocs.io/en/latest/index.html) for mocking. I prefer `mocker`'s fixture syntax over the built in `unittest.mock`, although the underlying concept is the same. If you're following along, you can `pip install pytest pytest-mock` inside your virtual environment, create a `test.py` file in the same directory as `get_github_user.py`, and paste the below code.

```python
import pytest
import requests

from get_github_user import get_github_user

def test_get_github_user(mocker):
    user = 'valinsky'
    mock_get = mocker.patch('get_github_user.requests.get')
    mock_get.return_value.status_code = requests.codes.ok
    mock_get.return_value.json.return_value = {'login': f'{user}'}

    json_response = get_github_user(user)

    mock_get.assert_called_once_with(f"https://api.github.com/users/{user}")
    assert json_response == {'login': f"{user}"}
```

The unit test is structured in 3 parts: variables setup and mocking, function call, assertions.

First, we mock the API call via `mocker.patch('get_github_user.requests.get')`. Now when the unit test is executed, `requests.get` will be a mock object, represented by `mock_get` in the test. In order to satisfy the happy path scenario, we configure `mock_get` with a `status_code` property of 200, and a `json()` method that returns a dictionary. Notice that we mocked the `requests.get` object, not the actual function call (the call parentheses `()` are missing). `mock_get` represents the function object. `mock_get.return_value` represents the object returned by the function after it was called. More on that below.

Second, we call `get_github_user` and capture the response.

Lastly, we assert that our mock was properly called and the function returns the expected response. Mocks offer you the ability to assert how they're called, if they're called, how many times they're called and with what parameters. For our use case, we simply need to assert our mock was called only once with the proper value.

Now if you run `pytest test.py` in your preferred terminal you should see the test passing.

Let's write our second test that covers the exception path.

```python
def test_get_github_user_exception(mocker):
    reason = 'Not Found'
    mock_get = mocker.patch('get_github_user.requests.get')
    mock_get.return_value.status_code = requests.codes.bad_request
    mock_get.return_value.reason = reason

    with pytest.raises(Exception) as e:
        get_github_user('valinsky')
    assert f"{e.value}" == reason
```

For this scenario, we want `get_github_user` to raise an exception. To achieve this, we configure our mock with a `status_code` != 200 and a `reason` property. We then assert that the exception was raised via `pytest.raises` and ensure the exception message is correct. If you run `pytest test.py` now you should see both tests passing.

### Diving deeper

Let's actually look at our mocks behind the scenes. We can add a breakpoint in our `get_github_user` function right after `requests.get` and run the first test.

``` python
...
response = requests.get(f"https://api.github.com/users/{user}")
breakpoint()
if response.status_code != 200:
...
```

The Python interpreter will now bring up the [pdb](https://docs.python.org/3/library/pdb.html) and we can inspect the mocks more in depth.

```shell
(Pdb) ll
4     def get_github_user(user: str) -> dict:
5         response = requests.get(f"https://api.github.com/users/{user}")
6         breakpoint()
7  ->     if response.status_code != requests.codes.ok:
8             raise Exception(response.reason)
9         return response.json()
(Pdb) requests.get
<MagicMock name='get' id='4356032032'>
(Pdb) requests.get()
<MagicMock name='get()' id='4360221664'>
(Pdb) response
<MagicMock name='get()' id='4360221664'>
(Pdb) response.status_code
200
(Pdb) response.json
<MagicMock name='get().json' id='4360221712'>
(Pdb) response.json()
{'login': 'valinsky'}
```

There multiple mocks at play here:
* `requests.get` is our main mock object. In our test it's represented by `mock_get`.
* `requests.get()` is a mock object as well, although a different object (denoted by a different id). It's represented by `mock_get.return_value` in our test. `response` is the same object.
* `response.status_code` is represented by `mock_get.return_value.status_code` in our test. This is not a mock since we explicitly assigned a value to it.
* `response.json` is a mock object, represented by `mock_get.return_value.json` in our test.
* `response.json()` is represented by `mock_get.return_value.json.return_value` in the test. Similarly to `response.status_code`, this is not a mock since we explicitly assigned a value to it.

Mocks can dynamically create other associated mocks without them being explicitly defined. If you try to get the value of an undefined property or method of a mock, the code will not break, instead it will return another mock.

```shell
(Pdb) response.random_property
<MagicMock name='get().random_property' id='4362565248'>
(Pdb) response.random_method()
<MagicMock name='get().random_method()' id='4361123120'>
```

Even though `random_property` and `random_method()` were not defined in our test, Python automatically returns new mocks when they are referenced as part of another mock. Notice we didn't explicitly define `mock_get.return_value.json` in our test, we only defined what the return value of that object being called is. No exception was raised when we tried to access it above, Python just went along with it. Mocks are cool like that. If `random_property` and `random_method()` were defined in our test to return a specific value, that value would've been returned instead of a mock, similar to `status_code` and `json()`.

### Conclusion

I remember this idea of mocks automatically returning newly created mocks for undefined references, and returning their respective values if the references were previously defined, was hard for me to wrap my head around for some reason. Like wtf are these magical dynamic objects that behave so weirdly? If you're in the same position I was in and are still struggling to understand them, I hope this article clears up at least some confusion. Feel free to ask questions below if mocking is still confusing to you.

It's important to understand mocking. It's a great tool for developers to test different scenarios and to ensure code correctness. As we've seen, they are highly versatile and can easily map to specific use cases. However, that versatility comes with a caveat. The testing scenario has to be clearly understood and mocks should be used properly. Mocks are used to simulate expected behaviors, not to make unit tests pass. It's pretty easy to manipulate a mock in a way that makes that annoying unit test pass. When used correctly they're very powerful and can significantly improve overall code quality.

Integration tests shouldn't use mocks. They are generally used in unit tests to mock dependencies. Not every dependency needs to be mocked though. I worked on multiple services that used a local database when running unit tests and the database calls were not mocked. It depends on the use case. Overall mocking is an invaluable tool and a must-know technique for every serious developer
