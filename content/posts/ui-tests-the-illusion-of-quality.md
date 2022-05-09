---
title: "The Illusion of Quality at the Cost of Quality"
description: UI tests are a very small part of a quality strategy and can create unnecessary divisions within a team. Avoid them.
date: '2022-05-09T09:00:00.000Z'
categories: ["testing", "agile"]
---

UI-based tests are often the de facto method of verifying software correctness in many organizations. It involves deploying the software, making sure all the integration points are hooked up, and then having people attack the UI using tests cases that were developed (often in isolation) to ensure the software meets specification. [Much has been written about](https://testing.googleblog.com/2015/04/just-say-no-to-more-end-to-end-tests.html) the downside of this approach which ends up resulting in slow and brittle tests. It is the main reason why [automation pyramids](https://martinfowler.com/bliki/TestPyramid.html) suggest that UI-based tests be the smallest number of tests written in a suite.

Beyond being brittle, unreliable and slow, the major problem with UI-based tests is that they simply don’t test enough combinations. Consider the following rather generic example of an architecture. It is a case where a browser interacts with the UI (written in a framework like React) and the React app calls a service layer which in-turn, calls other services or databases to eventually return a response to the user.

![/images/generic-architecture.jpg](/images/generic-architecture.jpg)

When a user performs an action (red line), there are seven different integrations that produce the results (green lines). It is along these green lines that most bugs exist, not on the red one. Yet we erroneously use the red line to instrument tests for the system. The goal is to attack the green lines efficiently while having a repeatable regression test suite.

The information a user submits can result in different combinations and variations of the green lines being invoked. In the best case scenario where we don’t care about order of calls or repetitive calls, the number of cases to test all combinations could be calculated using the “n choose k” approach:

$$
n!/k!(n-k)!
$$

| Number of green lines invoked with each user action | Number of tests needed |
| --- | --- |
| 1 | 7 |
| 2 | 21 |
| 3 | 35 |
| 4 | 35 |
| 5 | 21 |
| 6 | 7 |
| 7 | 7 |

The above table does not even account for variations between two components! For example, if one component sends a status code to another and returns a response, and there are five different status codes possible with five different responses, this would blow up the above counts by multiple orders of magnitude. 

Creating “prep data” for UI tests in these situations can be a months-long effort and can easily be avoided while increasing quality. For each test case we have to setup the right data in a deployed environment so we’re updating datastores that need to be constantly maintained which is a costly endeavour requiring inter-team communication (a great example of dependencies killing autonomy). If you have been through this exercise even once, you know it is a nightmare. It is unreasonable to think that this strategy can scale to beyond the simplest cases. And we haven’t even talked about speed and stability of these tests which will be woeful.

An alternate approach is to not rely on the UI to invoke the variations to test, but push them down to the actual components. Let’s consider this scenario which falls in the “Services” group in the above diagram (i.e., three different components to check each time, let’s not even worry about the UI components right now):

The user enters the number of US Dollars they want to convert to Bitcoin

- The system checks if the amount is greater than $1000 and if so, call a frauds checking service
- Award the user loyalty points (1 point for every $100 dollars converted), none for less than that
- Send a confirmation email if their settings have them enabled

Here is a reasonable workflow for testing the above:

- Capture the request/response payloads from the three different services *once* and store them as part of our test suite
- Hand-modify each payload to replicate different cases.
    - For example for the first requirement, mock the response from a service in the following cases to see if your calling component handles it appropriately:
        - Greater than $1000
        - Under $1000
        - Less than $1000
        - null
        - Empty string
        - Not a number
        - 10000000000000000000000000000
    - For the second requirement, it might be:
        - Greater than $100
        - Less than $100
        - $100
        - $0
    - For the third requirement, it might be
        - Email is present, setting is off
        - Email is present, setting is on
        - Email is not present, setting is off
        - Email is not present, setting is on
- Create “unit” level tests which invoke combinations of the above at blazing speed

Note that many of the tests cases above are not possible to run through the UI since UI-validation would prevent the services from being invoked, leaving services code potentially untested. In general, failure cases are harder to assert using the UI since the UI is designed to provide graceful experiences. Failures are often hidden behind more generic error messages so validating failure type is difficult through the UI but easy from the component level.

The above approach is not a technical challenge but an organizational one. Companies are wired to offload quality to specific roles like QA, who end up writing automation suites at the UI level where it’s impossible to provide the requisite quality. The better approach is to have QA work with developers and ask them “what if” questions which end up resulting in strong and comprehensive test suites. In the above example, QA might ask questions like:

- What if the user doesn’t have a USD bank account?
- What if the user has a previous fraudulent transaction?
- What if the services that provides the current value of Bitcoin is unavailable?

The developer can then use this information to simulate payloads and ensure the application has the right tests. This does not mean that UI tests go away, they are instead used to ensure the application is stable and running, rather than functionally correct. This reduces the UI tests needed from the hundreds to a handful. Exploratory testing ([PDF](https://less.works/papers/et.pdf)) is another method to identify quality issues from the user’s perspective, with output from exploratory testing used to inform what unit tests to write. 

This model of working requires QA to impart knowledge to developers, who then codify this knowledge as part of the codebase. It requires the two roles to work closely together rather than developers heaving code over the wall for QA to test, thus starting the long and painful process of fix-deploy-retest. The greater challenge here may simply be the mindset that UI tests are somehow more “real” than unit tests - they’re not. They provide the illusion of quality without poking into the details of how the various interconnected components work together to deliver a user experience. Most bugs live at the seams of a system and there is nothing better than unit tests to uncover them efficiently.