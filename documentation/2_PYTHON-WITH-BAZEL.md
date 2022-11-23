<h1>Working with Python and Bazel</h1>
Credits to:
https://www.youtube.com/watch?v=8P3m1-U7v0k&list=PLdk2EmelRVLovmSToc_DK7F1DV_ZEljbx&index=2
<br>

> <h2>Setting up Python for Bazel</h2>

- A [rule](https://bazel.build/extending/rules) defines a series of actions that Bazel performs on inputs to produce a set of outputs. 
- Currently the core rules are bundled with Bazel itself, and the symbols in this repository are simple aliases. However, in the future the rules will be migrated to Starlark and debundled from Bazel. Therefore, the future-proof way to depend on Python rules is via this repository (https://github.com/bazelbuild/rules_python). 

- To import rules_python in your project, you first need to add it to your WORKSPACE file:
~~~
load("@bazel_tools//tools/build_defs/repo:http.bzl", "http_archive")
http_archive(
    name = "rules_python",
    sha256 = "a868059c8c6dd6ad45a205cca04084c652cfe1852e6df2d5aca036f6e5438380",
    strip_prefix = "rules_python-0.14.0",
    url = "https://github.com/bazelbuild/rules_python/archive/refs/tags/0.14.0.tar.gz",
)
~~~

<br>

> <h2>Creating a simple Python library with Bazel</h2>

- As before, we need a folder and a BUILD. We'll add in a Python script as well as its test file.

~~~
mkdir projects/calculator && touch projects/calculator/BUILD.bazel %% touch projects/calculator/calculator.py && touch projects/calculator/calculator_test.py
~~~

- We want to populate the Python files. First, for calculator.py:
~~~
class Calculator:
    def add(self, x, y): return x + y
~~~
- Lastly, for calculator_test.py:
~~~
import unittest
from projects.calculator.calculator import Calculator

class TestSum(unittest.TestCase):
    def test_sum(self):
        calculator = Calculator()
        self.assertEqual(calculator.add(1, 2), 3)

if __name__ == '__main__':
    unittest.main()
~~~

- Then, we want to look up a nd use [Python Rules](https://bazel.build/reference/be/python). In our BUILD.bazel we will add the following two targets:
~~~
py_library (
    name = "calculator",
    srcs = ["calculator.py"],
)

py_test(
    name = "calculator_test",
    srcs = ["calculator_test.py"],
    deps = [
        "//projects/calculator:calculator",
    ],
)
~~~

<br>

> <h2>Testing Python libraries with Bazel</h2>

- We can run a suite of tests like this:
~~~
bazel test projects/calculator/...
~~~

- Note that we can also omit the '//' syntax. This allows us to make use of tab-autocomplete in the terminal. But ideally, let's keep it explicit with the full syntax.
- <b>Note:</b> If your tests fail (you can do this by changing 3 to 4 in the assertion), Bazel will give you a log of your failed test, which you can then read from its cache for more information on what went wrong. 

<br>

> <h2>Building a simple Flask Python App with Bazel</h2>

- Firstly, we need to define a new project. We can call it simply `flask-app`. It needs a `BAZEL.build` and a `main.py`:

~~~
from flask import Flask
from random import randint

from projects.calculator.calculator import Calculator

app = Flask(__name__)
my_calculator = Calculator()

@app.route('/')
def hello():
    num1, num2 = randint(0, 100), randint(0, 100)
    message = f"{num1} + {num2} = {my_calculator(num1, num2)}"
    
if __name__ == '__main__':
    app.run()
~~~

- Now for the targets, before we were using the `py_library` and the `py_test` rules. Instead, we now need to make use of the `py_binary` rule since we area actually going to be running the script:

~~~
py_binary (
    name = "main",
    srcs = ["main.py"],
    deps = ["//projects/calculator:calculator"]
)
~~~

- If you now try to run this binary executable, you will notice errors related to visibility. This is because we haven't actually made our `calculator` project to be visible to this `flask-app` project.
- We can do a couple of different things:
    - A) Set the visibility of `calculator` target (the library Python rule) to be <b>completely public</b>. The default is private.
    ~~~
    py_library (
        name = "calculator",
        srcs = ["calculator.py"],
        visibility = ["//visibility:public"]
    )
    ~~~
    - B) We can define custom visibility labels.


<br>

> <h2>Installing Python Third Party dependencies with PIP and Bazel</h2>

- If you try to run this now, you will notice the error related to visibility has vanished. But we still have a `ModuleNotFoundError: No module named 'flask'`. This is because if we don't have Flask installed (not by default with Python in any case), we can't run it. This is another dependency we need to add to our app, besides the calculator package.

- Firstly, we would like to add another folder at the top level (essentially another project):
~~~
mkdir third_party && touch third_party/BAZEL.build && touch third_party/requirements.lock
~~~

- You can install multiple dependency packages via requirements_loc.txt files and pip parse. We import Starlak's `pip_parse` and point it to the `my_deps` alias and its dependencies. In our case, we will write `Flask==2.2.2`. The following snippet goes into WORKSPACE.
~~~ 
load("@rules_python//python:pip.bzl", "pip_parse")

# Create a central repo that knows about the dependencies needed from
# requirements_lock.txt.
pip_parse(
   name = "my_deps",
   requirements_lock = "//third_party:requirements_lock.txt",
   python_interpreter = "python3"
)
# Load the starlark macro which will define your dependencies.
load("@my_deps//:requirements.bzl", "install_deps")
# Call it to define repos for your requirements.
install_deps()
~~~

-- We load the `requirement` function in projects/flask-app/BUILD.bazel, and with it we can add Flask as a deps:
~~~
load("@my_deps//:requirements.bzl", "requirement")

py_binary (
    name = "main",
    srcs = ["main.py"],
    deps = [
        "//projects/calculator:calculator",
        requirement("Flask")
    ]
)
~~~

- Then we want to build our repository again via `flask build //...`. BUT: as of Bazel 5.3.2, this will cause an error in the following code found in home/user/.cache/bazel/_bazel_user/some_sha256/external/my_deps_flask/BUILD.bazel (auto-generated artifact). More precisely, `Repository '@my_deps_werkzeug' is not defined and referenced by '@my_deps_flask//:pkg'`. We have a dependency added by us referencing its own deps, which we didn't resolve. The solution I found was to comment out the deps, as such:
~~~
py_library(
    name = "pkg",
    srcs = glob(["site-packages/**/*.py"], exclude=[], allow_empty = True),
    data = [] + glob(["site-packages/**/*"], exclude=["**/* *", "**/*.dist-info/RECORD", "**/*.py", "**/*.pyc"]),
    # This makes this directory a top-level in the python import
    # search path for anything that depends on this.
    imports = ["site-packages"],
    deps = ["@my_deps_click//:pkg","@my_deps_importlib_metadata//:pkg","@my_deps_itsdangerous//:pkg","@my_deps_jinja2//:pkg","@my_deps_werkzeug//:pkg"],
    tags = ["pypi_name=flask","pypi_version=2.2.2"],
)
~~~

- Now we can finally build and run our flask app. You can now access the link provided in the console and every time you refresh, you should see our calculator giving us different random sums.
~~~
bazel build //... && bazel run //projects/flask-app:main
~~~