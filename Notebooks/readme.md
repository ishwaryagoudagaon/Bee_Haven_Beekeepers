## Data-quality issues found and how they were solved

Every decision is backed by a chart in `01_Data_prep_sample_solution_annotated.ipynb`.

| Issue found                                                                | Decision                            | Why                                                                                                                         |
| -------------------------------------------------------------------------- | ----------------------------------- | --------------------------------------------------------------------------------------------------------------------------- |
| Timestamps stored as text, in German local time                            | Converted to **UTC**                | Daylight saving makes 02:00–02:59 happen twice in October; UTC gives every moment one timestamp and matches the weather API |
| Flow file = departures stacked on top of arrivals                          | Split into `flow_out` / `flow_in`   | One timestamp, one row, clear meaning                                                                                       |
| 697k "duplicate" rows in raw flow                                          | **Kept**                            | They are real zero counts (night) appearing once in each half; dropping them would delete data                              |
| Humidity of −100 % next to 100 %                                           | Absolute value                      | Sign-flip sensor error                                                                                                      |
| Weight of −170 g after the hive was emptied                                | Clipped to 0                        | A scale can't weigh less than nothing                                                                                       |
| Arrivals of 7,999 bees/min, and spikes at night                            | Set to NaN                          | Error code; more arrivals than the busiest departure minute ever is physically impossible                                   |
| IQR / z-score outlier rules                                                | Tested, **not used**                | Bee traffic is extremely skewed; they would flag \~175k normal busy minutes                                                 |
| Missing values and sensor outages                                          | Kept as NaN                         | Silver doesn't invent data; filling gaps is an analysis decision                                                            |
| Weather values borrowed from more distant stations (`fallback_source_ids`) | `<measure>_source_distance` columns | Analysts can see how far away each value was measured                                                                       |


