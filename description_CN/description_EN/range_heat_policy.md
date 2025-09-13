# RangeHeat Policy and Env

`RangeHeat` represents recent window heat used for display/sorting. Current code includes heat/time dirty processing; RangeHeat aggregation can be added by background jobs.

## Env
- `CHARACTER_RANGE_HEAT_WINDOW`: e.g., `7d`, `30d`, `1h`
- `CHARACTER_RANGE_HEAT_UPDATE_INTERVAL`: e.g., `1d`, `1h`
- `CHARACTER_MARKET_REFRESH_INTERVAL`: affects presign TTL and frontend list refresh cadence

## Notes
- Batch jobs should aggregate within the window and write `RangeHeat`
- Frontend may use `HeatScore` primary sort and `RangeHeat` secondary
