## Automated Testing Essential Concepts

It is absolutely critical that software be easy to modify and deploy. This is true for the person who wrote the code but is even more important when it comes to handing off your code to someone else. The person who writes the code is NEVER the ones who ends up maintaining it throughout its life.

The most important component of writing long lasting code that will avoid frustrating those that inherit it in the future is to write tests. Writing tests that are easy to run will help the person modifying the code to do so quick and with more confidence. Having tests available also decreases the chances that bugs will make it into production, thus improving the relationship with the user. There is little question about the utility of tests so we won't go deep into that. Instead we'll use this space to talk a bit about the different types of tests and when to use them.

## Unit tests VS integration tests

There are two main types of tests available to us. There is not a super clear boundary between the two yet there certainly is a spectrum. What we can do is use this section to outline some attributes of each. If you end up with tests that have some mix of the two, that's 100% okay! Sometimes the situation calls for it. However, you should be aware of the type of test you are looking at or writing in order to do your best to keep a clear demarkation.

That being said, let's start with our in-house definition of the two:

- **Unit test** - Tests a single function for correctness of input and the corresponding output. The function that you are testing may indeed call other functions in external libraries though they usually don't call other functions that are in the same codebase. Unit tests are about ensuring that low-level implementation details are rock solid.
- **Integration test** - Tests that a set of functionality works well in concert. These are necessary because if each of the units are covered and passing their unit tests, they still may be interacting with each other incorrectly and integration tests expose this. Integration tests are much more about ensuring that code generates the correct business value than low-level implementation details.

### Examples

1. A function that concats a password and salt and returns the hash would require unit tests.
1. An HTTP endpoint that uses the above function to authenticate a user would require an integration test.

This is because the function in (1) could be working perfectly whereas the endpoint code in (2) might be calling function (1) with the salt and username in reverse order by accident. In this view, the highest priority is to test that the login function works correctly which, implicitly, will be testing the most critical functionality of any functions it may be calling regardless of whether or not they were written by you.

## When to use which type of test

I generally use the following heuristic:

1. Write good integration tests first. Integration tests are all about ensuring business value is delivered and this is what the client pays for.
1. Write unit tests for either (a) super critical and complicated functions or (b) to reproduce a bug that you are working on. In either of these cases, you use TDD by writing code that explicitly declares the desired outcome and fix bugs until it does just that.

Some people disagree with focusing on integration tests over unit tests. What I've found is that unit tests can miss the most critical and obvious bugs. You can easily have each individual function working correctly while some of the code or infrastructure that glues it together has an obvious bug that, say, prevents an application from even being able to run. If you have a  test requires the application to be running then you'll ensure that the application can run. This is more valuable than ensuring that some edge case of some low level function that is not on the critical path is covered. Ideally you'd do both but in my experience, the integration tests need to be there from day 1 while a robust suite of unit tests tend to develop over time.

## Avoid integration test mocks

Something that has been very common in the past is to use mocks when testing in order to speed them up and remove the need to connect to external services. Although sometimes appropriate, mocks raise quite a few difficulties such as:

- The maintainer of the mock needs to maintain parity between their fake API and the real one. In the case of something like boto, that is very difficult.
- If the service is part of the functionality, make sure that works as well! If it is a mysql database, that's just fine. It makes the tests take longer but they bring more confidence. Usually you only run one of these at a time while you're working on it and if you have to wait for 5 seconds rather than 0.5 seconds for the tests to run it's usually a worthwhile tradeoff.

There are some situations (usually when writing unit tests rather than integration tests) in which you'll need to use a mock and that's okay. For example, AWS services like S3 or Kinesis firehose are difficult to have fully running, even for a full integration test. Rather, you should write simple wrappers for them yourself and mock those using the [core python mock](https://docs.python.org/3/library/unittest.mock.html) library rather than include a big dependency like moto.

## What to test

### When writing unit tests

You want to test code that you have written. Not code that others have written. Say, for argument sake, that you have the following function

```py
def _write_to_file(fname, string):
    with open(fname, 'wb') as fh:
        fh.write(string.encode())
```

In this example, you don't need to test much. This function isn't doing much except for calling another function. If you use it incorrectly, the program would crash on runtime because you forgot the `b` or forgot to `encode()` the string. As long as this function is called as part of an integration test somewhere, we can feel good about it. Some folks will disagree with me here and advocate that this should be treated like any other piece of functionality and tested but I don't find it's worth it to increase the codebase with these types of tests. Maybe time will change my mind though!

However, when you introduce logic, that is what needs to be tested:

```py
def _write_to_file(fname, string):

    if string is None:
        raise ValueError('string cannot be null')

    with open(fname, 'wb') as fh:
        fh.write(string.encode())
```

In this case, we've introduced some logic that you might want to test.

### When writing integration tests

For us, the most important type of integration test that we have are for our ETLs. Our ETLs are written in python and typically connect to a database where they execute SQL. In these cases, you should 100% leave a database running, let your automated tests connect to a database, do whatever setup or teardown on the database that is necessary, and test that your code actually interacts with the database correctly!

## Best practices

### Mig mama test

You should always have one comprehensive end-to-end test which tests the equivalent of the entire production system. These tests will be slow so you won't be able to write and use a LOT of them. But the ones you have will end up being the ones that help save you from releasing something that immediately bring down the entire system because of some simple oversight (e.g. mis-configuration).

### Bloated testing frameworks

Test are code. Code needs to be maintained. Having a poorly designed and bloated testing framework may actually end up slowing down releases without preventing that many bugs. This is one of the reasons that I, per unit of code, find integration tests bring a larger ROI than just unit tests.

## When testing ML systems

Some things change a bit when testing ML systems.

## When testing ETLs

Integration tests when writing pipelines are incredibly important and, even when running locally, should have at least a few of them that use the scheduling tech (such as digdag) that is used in production.

You should take great care when choosing your fixtures for tests. There are a few strategies possible when creating fixtures and they are:

1. To have lots of different fixtures. In this case, you'll create new fixtures for groups of tests rather than try to have one set of fixtures that are used across all of them.
1. To have few fixtures that are reused across the codebase. In this case, you choose to push the complexity to the codebase and keep the number of fixtures down.

When choosing (1), you'll end up spending more time selecting datasets and writing them to files. This will leave more non-code files in your repo (not necessarily bad as long as they are small). It will also require you to update more fixtures in the case that the input changes. Also, when you update fixtures, you'll probably need to update any test code that uses them as well.

When choosing (2) you're more likely to end up needing to change unrelated tests when a fixture changes. Imagine that you need to add a new row to a csv file which then causes a bunch of tests that have asserts having to do with numbers of rows to fail.

Both of these have different tradeoffs. If you don't expect the input data to change much, you might be better off going with (1) since you're less likely to have sweeping changes while also retaining the ability to tweak each of the tests to add edge and corner cases. If the input data is going to change much, it might be easier to choose (2) since you will spend less time making fixture changes when they do occur.

The tradeoff is a difficult one though and there's no clear option that gives you the best of both worlds that I know of. The most important thing is to recognize when you're creating a bloated test framework and re-assess.

## Python package
The package that we use for doing that is [pytest](https://docs.pytest.org/en/latest/index.html). You can just `pip install pytest` into your python environment, and you are going to be able to use it.

Normally, you should write the tests using the core python `unittest` library, but run the tests using `pytest ...`. It is not amazingly pretty, but it gets the job done.

## Exercises
An organization yearly applies a test to students from different schools in a city to measure their general level of knowledge. The organization hired us to process the results.

Each exam site has a log file where you can find the student's info: name, birthday, school, and the number of correct answers. All these log files are in a data directory, and they are the data source of your analysis. Remember to make sure that you are reading only the .csv files with the correct info because the machine that reads the scorecards is generating duplicated log files: one correct log file and one log file with one extra error column. The log files are [test_scores2.csv](./data/test_scores2.csv), [test_scores1.csv](./data/test_scores1.csv), [test_scores1_wrong.csv](./data/test_scores2_wrong.csv) and [test_scores2_wrong.csv](./data/test_scores2_wrong.csv).

Before doing the analysis, you should prepare the data because some features must be added to the dataset. First, you have to create their usernames. The username is the concatenation of **name + @ + school**. Then, you are going to need the age of the students because their score is based on an index inversely correlated to their ages. The target student group is 10-year-olds, and the exam is made for them. However, some younger and older students also take the test, and the index is higher and lower, respectively. So, to calculate the index, you need to apply the following formula: `index = 1 - ((age - 10) / 10)`. Finally, to calculate the score, you have to multiply the number of correct answers times the index, like `score = correct answers * index`.

Once you have the data prepared, you can start the analysis. The minimum score to get a student/school approved is more than 30 and the maximum is 100. It is a simple analysis where you should generate `csv` files with approved students, failed students, approved schools, and failed schools. These `csv's` should be in a results folder.

Based on the description above, do the following:

- Implement the problem presented in a structured manner (packages, modules, etc.).
- Perform a couple of great integration tests.
- Then write unit tests for the most difficult or critical functions.
- Suppose the client required a change and wants the __username__ to be on the following form: **name + / + school**. Then you should do:
    - Use TDD and write a unit test for the function that is being changes that will only pass when the changes are successfully made.
    - See that unit fail.
    - Fix the function.
    - See the unit test pass.
    - Write a unit test for ONLY the functions that they are changing to ensure that the changes are working correctly.

Remember to create a repo for this exercise and commit each step, so we will know you went through everything.