<h1>Getting started with Bazel</h1>
Credits to:
https://www.youtube.com/watch?v=BZYj6yfA6Bs&list=PLdk2EmelRVLovmSToc_DK7F1DV_ZEljbx&index=1
<br>

> <h2>How to install Bazel?</h2>

- Install homebrew

~~~
/bin/bash -c "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/HEAD/install.sh)"
~~~
- Install bazelisk with brew and follo instructions to add it to PATH.

~~~
brew install bazelisk
~~~

- Check installation

~~~
bazel --version
~~~

<br>

> <h2>How to build a project with Bazel?</h2>

- Create files we need for a bazel project. Before you can start a build, you need a workspace. A workspace is a directory tree that contains all the source files needed to build your application.

~~~
touch .bazelversion && touch WORKSPACE.bazel && touch BUILD.bazel
~~~

- Syntax to build the project. '//' means we are specifying the project path as the current working directory. '...' means we want to build simply everything under our specified project path. 

~~~
bazel build //...
~~~

- This will have created several folders:
    - bazel-bin
    - bazel-repo-name (in our case '<i>multi-language-bazel-monorepo</i>')
    - bazel-out
    - bazel-testlogs

<br>

> <h2>Let's simulate having a monorepo with several projects:</h2>

- First, let's create a folder structure with multiple projects for our monorepo. bazel documentation will refer to these as `packages`.

~~~
mkdir projects && mkdir projects/projectA && mkdir projects/projectB
~~~

- Then, let's create a BUILD.bazel file for each respectively. Let's also remove the original BUILD file we used to demonstrate the build action earlier. 
~~~
touch projects/projectA/BUILD.bazel && touch projects/projectB/BUILD.bazel && rm BUILD.bazel
~~~

<br>

> <h2>Building a specific project or several at once:</h2>

- In order to build just ProjectA
~~~
bazel build //projects/projectA/...
~~~

- In order to build just both A and B
~~~
bazel build //projects/...
~~~

<br>

> <h2>Adding an actual target to our build:</h2>

- Use a ['genrule'](https://bazel.build/reference/be/general#genrule) to create a hello world file. For this, go to the BUILD.bazel of ProjectA and paste the following.
~~~
genrule(
    name = "hello",
    outs = ["hello_world.txt"],
    cmd = "sleep 5 && echo 'Hello World!' > $@"
)
~~~

- Now build the entire monorepo. You will notice because of sleep 5, it takes just over 5 seconds to complete the build. But, if we build it again, Bazel's cache will allow this to be quick, as nothing really changed. If we were to add more exclamation marks to the 'Hello World!' text, it will take again over 5s.
- If you're wondering where the hell is our hello_world.txt file, the output of the build will go into `bazel-bin/projects/projectA/hello_world.txt`.
- These files would generally be in your `.gitignore` and not pushed to source control.
~~~
bazel build //...
~~~

- Let's add a script to ProjectB's BUILD.bazel. This will also sleep for 5 seconds. Build the project again. You will notice that it's still around 5s runtime, not 10. This is because Bazel builds A and B in parallel.
~~~
genrule(
    name = "mom",
    outs = ["hello_mom.txt"],
    cmd = "sleep 5 && echo 'Hello Mom!' > $@"
)
~~~
- If you use sleep for 7s, the build will take around 7s total.


<br>

> <h2>Building a specific target.</h2>

- Say we add another target to one of our packages (let's say ProjectA). Just append it to the BUILD.bazel file, as such:
~~~
genrule(
    name = "greetings",
    outs = ["greetings_terra.txt"],
    cmd = "echo 'Greetings, Terra!' > $@"
)
~~~

- Now we can choose to only build this target, using the target name and the following syntax:
~~~
bazel build //projects/projectA:greetings
~~~