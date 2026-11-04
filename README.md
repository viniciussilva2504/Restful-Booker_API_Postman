# Restful-Booker API tests

Postman collection and Newman runner for the public [Restful-Booker](https://restful-booker.herokuapp.com) API ([apidoc](https://restful-booker.herokuapp.com/apidoc/index.html)).

Vinicius Jesus da Silva

[![API tests](https://github.com/viniciussilva2504/Restful-Booker_API_Postman/actions/workflows/api-tests.yml/badge.svg)](https://github.com/viniciussilva2504/Restful-Booker_API_Postman/actions/workflows/api-tests.yml)

The badge tracks the `API tests` workflow on the default branch. It has no status until that workflow file is on `main`.

## What is tested

The collection calls the live API. It creates a booking with a unique name and uses that id for the read, filter, update, and delete steps.

Every request checks the HTTP status and the response time (under 20 seconds). JSON bodies are checked with `pm.response.to.have.jsonSchema`. Create, read, and update requests also check field values against the payload from this run.

The response-time check is a single collection-level test, so it runs on every request. `POST /auth` saves the token in its test script. Later updates and deletes send `Cookie: token=<value>`. A collection pre-request also requests a token when a folder is run on its own. That is how the data-driven folder authenticates.

`data/bookings.csv` has 5 booking payloads. `npm run test:data` runs the `DataDriven` folder once per row: create, then delete.

## Endpoints and cases

| Endpoint | Cases in the collection |
| --- | --- |
| `GET /ping` | 201, body `Created`, `Content-Type` contains `text/plain` |
| `POST /auth` | A valid login returns a 15-character hex token and saves it. A wrong password and an empty body both return 200 with `reason` `Bad credentials`. |
| `POST /booking` | Create echoes the payload (200). An incomplete body returns 500. Malformed JSON returns 400. An unparseable check-in is returned as `0NaN-aN-aN`. A negative price and a checkout before check-in are stored as sent. Five CSV rows are created and then deleted. |
| `GET /booking/:id` | The created booking, the booking after updates, 404 after delete, 404 for `does-not-exist`, and 418 when `Accept` is `text/html` |
| `GET /booking` | `firstname` plus `lastname` includes this run's id. A check-in/check-out window one day wider than the stay also includes it. The exact check-in date returns no rows for that guest. |
| `PUT /booking/:id` | 403 with no token, 200 when the full body is replaced, 400 for an incomplete body, 405 for an unknown id |
| `PATCH /booking/:id` | 403 with no token, a partial body changes only the sent fields, the same call succeeds with HTTP Basic, 405 for an unknown id |
| `DELETE /booking/:id` | 403 with no token, with an unknown token, and with `Authorization: Bearer`. A cookie token returns 201. An unknown id returns 405. Quirk rows and CSV rows are deleted in the same run. |

## Suite size

Counted from `postman/restful-booker.postman_collection.json`:

- **34 requests**
- **87 assertions** (`pm.test` calls): 86 on individual requests and 1 collection-level response-time check

`npm test` runs two Newman commands. These scripts do not skip requests, so the assertion totals are fixed:

| Command | Collection requests | Assertions |
| --- | --- | --- |
| `npm run test:api` (Health, Auth, Booking, Negative) | 32 | 113 |
| `npm run test:data` (`DataDriven`, 5 CSV rows) | 10 (2 requests × 5 rows) | 35 |
| Total | 42 | 148 |

The 113 assertions are 81 request-level tests plus the response-time check on each of the 32 requests. The 35 assertions are 5 request-level tests × 5 rows, plus the response-time check on each of the 10 requests.

The HTML summary shows 33 and 11 requests. The extra request on each command is the collection pre-request `POST /auth`, which Newman counts separately from the collection items.

## Run locally

CI and `.nvmrc` use Node 26, the current LTS line. Direct dependencies are pinned to Newman 6.2.3 and newman-reporter-htmlextra 1.23.1. `package-lock.json` locks the rest of the tree.

```bash
npm ci
npm test
```

`npm run test:api` and `npm run test:data` run the two halves on their own.

`postman/restful-booker.postman_environment.json` sets `baseUrl` to `https://restful-booker.herokuapp.com` and the public demo user `admin` / `password123`. The repo has no other credentials.

## Report

A local `npm test` writes:

- `reports/newman-report.html`
- `reports/data-driven-report.html`

JSON summaries are written beside them (`newman-report.json`, `data-driven-report.json`). Report output is gitignored.

## CI

`.github/workflows/api-tests.yml` runs on `push`, `pull_request`, every day at 12:00 UTC, and `workflow_dispatch`. The job installs Node 26, runs `npm test`, and uploads both HTML reports and both JSON summaries as the `newman-html-report` artifact, kept for 14 days. Download it from the workflow run.

GitHub Pages is not used. A Pages deploy from Actions needs the repository setting Pages → Source: GitHub Actions, and this workflow's token cannot turn that setting on.

## Findings

Checked against the live API on 4 November 2026. The handlers cited here are in the published server ([mwinteringham/restful-booker](https://github.com/mwinteringham/restful-booker), `routes/index.js` and `helpers/parser.js`). The collection asserts this behaviour, and the request scripts comment each quirk.

- `GET /ping` returns **201** and the body `Created` (`res.sendStatus(201)`).
- `POST /booking` returns **200** on success (`res.send`).
- `DELETE /booking/:id` returns **201** and the body `Created`.
- `POST /auth` with a wrong or missing password returns **200** and `{"reason":"Bad credentials"}`. That branch does not set a status code.
- Unknown ids on `PUT`, `PATCH`, and `DELETE` return **405**. `GET` of an unknown id returns **404**.
- A validation failure on `POST /booking` returns **500**. An incomplete `PUT` body returns **400**. Required fields are `firstname`, `lastname`, `totalprice`, `depositpaid`, `bookingdates.checkin`, and `bookingdates.checkout`. `additionalneeds` is optional. One CSV row leaves it empty, and the test expects the property to be absent.
- Malformed JSON returns **400**.
- Date filters are `$gt` on `new Date(checkin).toISOString()` and `$lt` on `new Date(checkout).toISOString()`. The apidoc text describes both as greater-than-or-equal. Stored dates are `YYYY-MM-DD`, which sort before `YYYY-MM-DDT00:00:00.000Z`, so a check-in query equal to the stored day matches nothing. A window from the day before check-in through the day after check-out matches. Both filters are combined with the unique first name from the run.
- An unparseable check-in is accepted. `date.format(new Date(value), "YYYY-MM-DD")` renders an invalid date as `0NaN-aN-aN`.
- A negative `totalprice` and a checkout before check-in are stored as sent. The validator only checks that the fields are present.
- `Accept: text/html` on `GET /booking/:id` returns **418** and `I'm a Teapot`. The parser allows `application/json`, `application/xml`, `application/x-www-form-urlencoded`, and `*/*`. JSON requests in this suite send `Accept: application/json`. A probe with `Accept: application/xml` returned XML. That response is not asserted in the collection.
- Mutations accept `Cookie: token=<value>`, or `Authorization` equal to `Basic` plus the base64 of `admin:password123`. `Authorization: Bearer <token>` returns **403**. The Basic value is built in the collection pre-request from the environment user and password.
- The published server keeps tokens in a process-local `globalLogins` map and bookings in a LokiJS collection. The project homepage says the data resets about every 10 minutes. This suite stays inside one Newman process and only uses ids it just created.

## Layout

```
postman/restful-booker.postman_collection.json
postman/restful-booker.postman_environment.json
data/bookings.csv
.github/workflows/api-tests.yml
```
