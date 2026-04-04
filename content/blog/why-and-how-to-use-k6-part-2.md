---
title: "Why and how to use k6? - Part 2"
date: 2024-02-20
description: "A practical guide to structuring k6 load test projects - covering folder layout, writing main.js with options, setup, and report generation, and building reusable microservice controller files with custom metrics."
draft: false
---

In the last [article](/blog/why-and-how-to-use-k6/), we discussed a few introductory topics that will be helpful to get started with k6. Today, we are going to discuss how to write and structure the tests.

### How to structure tests?

You can either write everything in a single file or create multiple files depending on the number of microservices. Here's how I do it:

- Create a `main.js` file at the root of the project folder. This file should include running configurations, report imports and methods, setup functionality, constants, and the main function.
- Keep the rebuilt k6 executable file at the root of the project folder; you can ignore it during the testing process.
- Create a folder for microservices, named `controllers` at the root of the project folder. This folder will contain multiple `.js` files for different microservices, depending on how you have separated them.
- Each `.js` file inside the controllers' folder can contain a single or multiple microservices.
- Create a folder for reports at the root of the project.
- Create a folder to store all the required test data.

This is pretty much everything you need to do.

### Writing the tests

k6 only supports JavaScript as of now. Let's start with `main.js`. Sharing a sample below:

```js
// importing all the required modules from k6
import { check, fail, group, sleep } from "k6";
import http from "k6/http";

// import the required files for generating PDF reports
import { textSummary } from "https://jslib.k6.io/k6-summary/0.0.1/index.js";
import { htmlReport } from "https://raw.githubusercontent.com/benc-uk/k6-reporter/main/dist/bundle.js";

// import controllers
import { getMenu } from "./controllers/menu.js";
import {
  getGuestProfile,
  searchGuestProfile,
} from "./controllers/guestProfile.js";

// Set environment
const env = "xs";

// Set other constants
const returnReport = true;
const baseUrl = `https://test.${env}.com`;
const authUrl = `https://test-auth.${env}.com`;

export const options = {
  // Runs the load test with the same number of VUs for a specific duration
  vus: 750,
  duration: "15m",

  // Runs the load test in stages with a different number of Virtual Users (VUs) and durations
  // stages: [
  //   { duration: "20s", target: 10 },
  //   { duration: "30s", target: 100 },
  //   { duration: "3m", target: 200 },
  // ],

  // Checks and thresholds for the load test
  thresholds: {
    // Default checks
    checks: ["rate>0.9"],

    // Success and failed request counts and checks
    // failedGetMenuRequestCount: ["count <= 10"],
    // successGetMenuRequestCount: ["count >= 10"],
    // failedGetGuestProfileRequestCount: ["count <= 10"],
    // successGetGuestProfileRequestCount: ["count >= 10"],
    // failedSearchGuestProfileRequestCount: ["count <= 10"],
    // successSearchGuestProfileRequestCount: ["count >= 10"],

    // Overall API timings and checks
    apiTimings_getMenu: ["p(95) < 2000"],
    apiTimings_getGuestProfile: ["p(95) < 2000"],
    apiTimings_searchGuestProfile: ["p(95) < 2000"],
  },
};

// Function to do setup befor starting load test
export function setup() {
  const res = getToken();
  if (res.status !== 200) fail("Failed to get the auth token");
  const token = JSON.parse(res.body).access_token;
  return { token };
}

// Function to generate token
export function getToken() {
  const headers = {
    Authorization: "Basic test",
    "Content-Type": "application/x-www-form-urlencoded",
    "Cache-Control": "no-cache",
  };
  const payload = {
    grant_type: "access",
    scope: "roles",
  };
  const url = `${authUrl}/oauth2/access_token`;

  const res = http.post(url, payload, { headers: headers });

  return res;
}

// This function returns summary and generate reports
export function handleSummary(data) {
  const stdout = textSummary(data, {
    indent: " ",
    enableColors: returnReport,
  });

  const testResult = `reports/${env}testResult${
    new Date().toISOString().replace(/[:]/g, "_").split(".")[0]
  }.html`;

  return returnReport
    ? {
        [testResult]: htmlReport(data),
        // "reports/testResult.json": JSON.stringify(data, null, 2), //Uncomment this line if you require the result in a JSON file.
        stdout,
      }
    : {
        stdout,
      };
}

// main function
export default function (data) {
  const res = http.get(`${baseUrl}/actuator/health`);
  check(res, {
    appIsHealthy: (res) => JSON.parse(res.body).status === "UP",
  });
  group("Menu", () => {
    getMenu(baseUrl, data.token); //dependent on meal period
  });
  group("Guest profile", () => {
    getGuestProfile(baseUrl, data.token);
    searchGuestProfile(baseUrl, data.token);
  });
  sleep(1);
}
```

Let's discuss a few things done in the above code:

1. Start by importing all the required modules from k6.
2. Then import any external dependencies that are required.
3. Next, import the microservices created in the controllers, which will essentially be functions as well.
4. Set the constants such as environment variables, API URLs, and any other necessary details.
5. Define the `const options`, which is a default from k6. It will contain the running configurations, checks, thresholds, and any other required custom details.
6. There are two ways to run the test: one is in stages and the other is with a constant number of virtual users. Both options are shared above; choose as per your needs.
   - You can add as many stages as you want. There are no restrictions, just make sure you have enough resources to run them.
7. Then, there's the `setup` function, which runs before starting the main function. It's a good point to obtain access tokens and other details needed for the tests.
8. After that, we have the `handleSummary` function, which is responsible for storing the test reports. In this function, a small function is added to get the report name with a timestamp, which helps when sorting through multiple reports.
9. Finally, there's the main function, responsible for running the complete tests.
   - Within this function, the code checks for the service's health before making calls to different microservices.
   - It's important to verify if the service is reachable before making requests, as the purpose of the test is to determine how much load it can handle.
   - If the service is reachable, it proceeds to make calls to the microservices and updates the report.
10. Sleep functions are used to wait for a second before making the next request once the previous request is completed.

This is pretty much everything about `main.js`. If you need more information about any function, you can refer to the official docs.

Now let's take a look into the microservice file. I'm sharing a sample below:

```js
// importing all the required modules
import { URLSearchParams } from "../dependencies/URLIndex.js";
import { check } from "k6";
import http from "k6/http";
import { Trend, Counter } from "k6/metrics";
import { getvenueId } from "../testdata/readData.js";

// Custom metrics
const failedRequestCount = new Counter("failedGetMenuRequestCount");
const successRequestCount = new Counter("successGetMenuRequestCount");
const getTrend = new Trend("apiTimings_getMenu");

// Method to send getMenu requests
export function getMenu(baseUrl, token) {
  const searchParams = new URLSearchParams([
    ["place", `${getvenueId()}`],
    ["category", "food"],
  ]);

  const res = http.get(`${baseUrl}/menu?${searchParams}`, {
    headers: {
      Authorization: `Bearer ${token}`,
      Accept: "/",
    },
  });

  const result = check(res, {
    getMenu: (res) => res.status === 200,
  });

  failedRequestCount.add(!result);
  successRequestCount.add(result);
  getTrend.add(res.timings.duration);
}
```

Let's discuss a few things done in the above code:

1. Start by importing all the required modules.
2. Create custom metrics for each API. By default, the final result will contain consolidated stats of all the APIs, but if you need to see the stats of each API separately, you can add similar custom metrics.
3. Create a function for each API. It will make the call, check if the returned response is successful or not, and then update the custom metrics accordingly.

That's all you need to do for other microservices as well, and you are good to go.

Similarly, you can add more APIs and then just import them into the `main.js` and add them to the main function. No need to change any configuration. This will make your test template easily extensible.

I hope it helps, and that's all for today. Next time, we'll talk about how to randomize and access the data. Until then, have a great time!
