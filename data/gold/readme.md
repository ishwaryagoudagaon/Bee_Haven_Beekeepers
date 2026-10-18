## Gold layer: analysis-ready tables

| Table                     | Grain                                 | Content                                                                                            |
| ------------------------- | ------------------------------------- | -------------------------------------------------------------------------------------------------- |
| `hivedata_hourly` ★       | 1 row per hour per hive               | Departures, arrivals, hive temperature/humidity/weight + all weather measures, aligned on UTC hour |
| `hivedata_daily`          | 1 row per day                         | Daily totals (bees, rain, sunshine hours) and averages, plus **daily weight change**               |
| `activity_by_time_of_day` | morning / afternoon / evening / night | Average bees per hour                                                                              |
| `activity_by_temperature` | 5 °C outside-temperature bands        | Daytime bees per hour                                                                              |

Why **hourly**: the sensors tick at different rates (1 min, 5 min, 12 h) and weather is hourly. Joining on exact timestamps leaves
most cells empty, so aggregating to hours gives every row its matching weather. Counts are **summed**, measurements **averaged**.

- Bee traffic rises steeply once the hive is above \~20 °C (correlation +0.66 with hive temperature).
- The hive gets **drier on busy days** (−0.74), likely because bees fan their wings to dry nectar into honey.
- Weight is a running total, so the **daily change** in weight, not its level, is what relates to activity.
- The colony keeps the brood nest at \~35 °C all summer, regardless of the weather outside.
