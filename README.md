# JMeter Performance Testing

Load tests I designed and ran with **Apache JMeter** on three applications: two e-commerce web stores and a fleet-management system with both a web portal and a mobile (iOS) REST API. Each scenario was run at increasing numbers of concurrent users to find where response times start to degrade.

> The client names, hosts, user accounts and tokens have been removed. Hosts and credentials are now JMeter properties, so the scripts can be pointed at any environment.

## Tools & techniques

- Apache JMeter: Thread Groups, HTTP Request samplers, HTTP Request Defaults, Header and Cookie Managers, Loop and Transaction Controllers, User Defined Variables
- Scripts recorded with the HTTP(S) Test Script Recorder, then cleaned up: third-party and analytics calls removed, values parameterized
- Step load testing: 1 → 5 → 10 → 20 → 50 → 100 → 200 → 300 users
- Analysis with the Aggregate Report: average, median, 90th/95th percentile, error %, throughput

## Repository structure

```
scripts/        JMeter test plans (.jmx)
results/        Aggregate Report for each scenario and user load (<scenario>_<NNN>-users.csv)
results_summary.csv   One row per run (TOTAL line of each aggregate report)
```

| Application | Scenario | Script | User loads tested |
|---|---|---|---|
| Fashion e-commerce store (Shopify + GraphQL API) | Login | `fashion-store_login.jmx` | 1, 5, 10, 20 |
| | My Profile | `fashion-store_my-profile.jmx` | 1, 5, 10, 20 |
| | Add to cart | `fashion-store_add-to-cart.jmx` | 1, 5, 10, 20 |
| Fleet management – web portal (ASP.NET) | Login | `fleet-portal_login.jmx` | 5 – 100 |
| | Home / dashboard | `fleet-portal_home-page.jmx` | 5 – 300 |
| | Fleet view | `fleet-portal_fleet-view.jmx` | 10 – 100 |
| Fleet management – iOS app REST API | Login → Home → Fleet info | `fleet-mobile-api_ios.jmx` | 10 – 200 |
| Coffee e-commerce store (Magento) | Home, Login, Main menu | results only | 10, 20, 50 |

## How to run

```bash
jmeter -n -t scripts/fleet-mobile-api_ios.jmx \
  -Jthreads=50 -Jrampup=50 \
  -JAPI_HOST=api.your-env.com -JUSERNAME=test@your-env.com -JPASSWORD=*** \
  -l results/run.jtl -e -o results/html-report
```

- `threads` sets the number of virtual users, and `rampup` sets how many seconds it takes to start them all.
- `-l` saves the raw results. `-e -o` builds JMeter's HTML dashboard.
- Run in non-GUI mode (`-n`) for real load. Use the GUI only to build and debug scripts.

## Key results

Response times are in milliseconds. Full data is in `results_summary.csv`.

**Fleet mobile API (iOS)**

| Users | Avg | 90th % | Error % | Throughput/s |
|---|---|---|---|---|
| 10 | 1271 | 2369 | 0% | 9.1 |
| 50 | 1341 | 2488 | 0% | 10.5 |
| 100 | 2187 | 4226 | 0% | 11.2 |
| 200 | 1938 | 3499 | 0% | 11.8 |

- The API was stable, with no errors up to 200 users.
- Throughput levelled off at about 11–12 requests/s from 50 users onward. That is the server's capacity limit, so adding more users only adds waiting time.

**Fleet web portal – Home page**

| Users | Avg | 90th % | Error % |
|---|---|---|---|
| 20 | 2988 | 7551 | 37.5% |
| 100 | 5569 | 15182 | 37.5% |
| 300 | 25743 | 82913 | 37.7% |

- Response time degraded sharply above 200 users. The 90th percentile rose from about 15 s to 83 s.
- The error rate stayed flat at every load, so the errors came from the script, not from load: the CAPTCHA and ASP.NET `__VIEWSTATE` values were recorded, not correlated. See "Next improvements."

**Fashion store**

| Scenario | 10 users avg / 90th % | 20 users avg / 90th % |
|---|---|---|
| My Profile | 2924 / 1687 | 10323 / 16681 |
| Add to cart | 2526 / 1673 | 7652 / 8480 |

- Performance held steady up to 10 users and degraded at 20 users. The 90th percentile was about 10× higher at 20 users.
- Most errors in these runs came from third-party tracking calls, which have since been removed from the scripts.

**Coffee store**

- At 50 users, throughput fell (main menu: 8.9 → 2.1 requests/s) and the 95th percentile exceeded 190 s.
- The site saturates somewhere between 20 and 50 concurrent users.

## Next improvements

- Correlate dynamic values with a Regular Expression or JSON Extractor instead of hard-coded recorded values. This covers `__VIEWSTATE`, `__EVENTVALIDATION`, and auth tokens.
- Load test accounts from a CSV Data Set Config, so each virtual user logs in as a different user.
- Add Response Assertions and Think Times (Uniform Random Timer) for more realistic load.
- Run from CI (GitHub Actions) and publish the HTML dashboard report.

---
Author: **Sugandha Sinha** · [LinkedIn](https://www.linkedin.com/in/sugandha-sinha-b158062b) · [GitHub](https://github.com/SugandhaSinha07)
