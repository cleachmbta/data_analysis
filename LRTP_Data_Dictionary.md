# LRTP Data Dictionary

## Primary VehiclePositions Table
### Custom Calculations
Field | Description | Field Type | Query |
--- | --- | --- |  --- |
Trip Service Date | Format `vehicle.trip.start_date` as a Date | Date | `date(DATEPARSE("yyyyMMdd", [vehicle.trip.start_date]))`
VehiclePositions Unique Daily Trip Identifier | Field to uniquely identify trip departures based on Trip Service Date and Trip ID | String | `str([Trip Service Date]) + " " + [vehicle.trip.trip_id]`
Departure Time | The time that a trip departs (the first time that the vehicle starts moving towards the following station) | Date & Time | `{ FIXED [VehiclePositions Unique Daily Trip Identifier]: MIN([VehiclePositions feed_timestamp] END)}`
Departure Stop ID | The stop ID of the station that a trip is departing to | String | `{fixed [VehiclePositions Unique Daily Trip Identifier]: min(if [VehiclePositions feed_timestamp] = [Departure Time] then [vehicle.stop_id] END )}`
Route | ... | String | `{fixed [VehiclePositions Unique Daily Trip Identifier]: min(if [VehiclePositions feed_timestamp] = [Departure Time] then [vehicle.trip.route_id] END )}`
Vehicle Consist | ... | String | `{fixed [VehiclePositions Unique Daily Trip Identifier]: min(if [VehiclePositions feed_timestamp] = [Departure Time] then [vehicle.vehicle.label] END )}`
Trip Terminal | ... | String | `CASE [Departure Stop ID] WHEN "70502" THEN "Union Square" WHEN "70510" THEN "Medford/Tufts" WHEN "70110" THEN "Boston College" WHEN "70236" THEN "Cleveland Circle" WHEN "70162" THEN "Riverside" WHEN "70274" THEN "Mattapan" END`
Day of Week | ... | String | `DATENAME('weekday', [Trip Service Date])`
Hour | ... | String | `IF (DATEPART('hour',[Departure Time]))=0 THEN '12AM' ELSEIF (DATEPART('hour',[Departure Time]))=12 THEN '12PM' ELSEIF (DATEPART('hour',[Departure Time]))>12 THEN STR((DATEPART('hour',[Departure Time]))-12) + 'PM' ELSE STR((DATEPART('hour',[Departure Time]))) + 'AM' END`
Trip Departure Rank per Terminal | ... | Number (whole) | `{PARTITION [VehiclePositions Unique Daily Trip Identifier]: {ORDERBY [Departure Time] ASC,[vehicle.trip.trip_id] ASC: RANK_DENSE() }}` |

### Data Filters
- `vehicle.trip.revenue`=TRUE
- `vehicle.trip.trip_id`!=NULL
- `vehicle.stop_id`!=NULL
- `feed_timestamp`!=NULL
- `Departure Time`!=NULL

---

## Secondary VehiclePositions Table
### Custom Calculations
Field | Description | Field Type | Query |
--- | --- | --- |  --- |
Next Trip Departure Rank per Terminal | ... | Number (whole) | `[Trip Departure Rank per Terminal]-1`
Next Vehicle Consist | ... | String | Rename `[Vehicle Consist]`
Next Trip Terminal | ... | String | Rename `[Trip Terminal]`
Next Trip Service Date | ... | Date | Rename `[Trip Service Date]`
Next Departure Time | ... | Date & Time | Rename `[Departure Time]` |
- Left join the primary VehiclePositions table and Secondary VehiclePositions tables on `Trip Departure Rank per Terminal`=`Next Trip Departure Rank per Terminal`, `Trip Service Date`=`Next Trip Service Date`, and `Trip Terminal`=`Next Trip Terminal`

---

## Tertiary VehiclePositions Table
### Custom Calculations
Field | Description | Field Type | Query |
--- | --- | --- |  --- |
Trip Departure Rank per Terminal After Next | ... | Number (whole) | `[Trip Departure Rank per Terminal]-2`
Vehicle Consist After Next | ... | String | Rename `[Vehicle Consist]`
Trip Terminal After Next | ... | String | Rename `[Trip Terminal]`
Trip Service Date After Next | ... | Date | Rename `[Trip Service Date]`
- Left join the primary VehiclePositions table and Tertiary VehiclePositions tables on `Trip Departure Rank per Terminal`=`Trip Departure Rank per Terminal After Next`, `Trip Service Date`=`Trip Service Date After Next`, and `Trip Terminal`=`Trip Terminal After Next`

---

## Quaternary VehiclePositions Table
### Custom Calculations
Field | Description | Field Type | Query |
--- | --- | --- |  --- |
Prior Trip Departure Rank per Terminal | ... | Number (whole) | `[Trip Departure Rank per Terminal]+1`
Prior Vehicle Consist | ... | String | Rename `[Vehicle Consist]`
Prior Trip Terminal | ... | String | Rename `[Trip Terminal]`
Prior Trip Service Date | ... | Date | Rename `[Trip Service Date]`
-	Left join the primary VehiclePositions table and Quaternary VehiclePositions tables on `Trip Departure Rank per Terminal`=`Prior Trip Departure Rank per Terminal`, `Trip Service Date`=`Prior Trip Service Date`, and `Trip Terminal`=`Prior Trip Terminal`

---

## Union of all VehiclePositions tables
- Do a union of all VehiclePositions tables

### Custom Calculations
Field | Description | Field Type | Query |
--- | --- | --- |  --- |
Number of Cars | The total number of cars in a trip departure's vehicle consist | Number (whole) |  `{ FIXED [VehiclePositions Unique Daily Trip Identifier]: MAX(IF(LEN([Vehicle Consist])=4) THEN 1 ELSEIF (LEN([Vehicle Consist])=9) THEN 2 END)}` |
Vehicle Consist Backwards | If there are 2 cars in a trip's vehicle consist then swap their order | String | `if([Number of Cars]=2) then (RIGHT(STR([Vehicle Consist]), 4) + "-" + LEFT(STR([Vehicle Consist]), 4)) else [Vehicle Consist] end`
Gap to Next Terminal Departure | ... | Number (whole) | `{ FIXED [VehiclePositions Unique Daily Trip Identifier]: MAX(DATEDIFF('minute',[Departure Time],[Next Departure Time]))}`

---

## TripUpdate Table
### Custom Calculations
Field | Description | Field Type | Query |
--- | --- | --- |  --- |
Prediction Service Date | Format `trip_update.trip.start_date` as a Date | Date | `date(DATEPARSE("yyyyMMdd", [trip_update.trip.start_date]))`
TripUpdate Unique Daily Trip Identifier | Field to uniquely identify trip departures based on trip service date and trip ID | String | `str([Prediction Service Date]) + " " + [trip_update.trip.trip_id]`
TripUpdate Unique Prediction ID | Field to uniquely identify predictions based on the generated prediction time, the predicted departure time, and trip ID | String | `str([TripUpdate feed_timestamp]) + " " + str([trip_update.stop_time_update.departure.time]) + " " + [trip_update.trip.trip_id]`
Predictions After Departure Time | ... | Boolean | `IF (DATEDIFF('second',[TripUpdate feed_timestamp],[trip_update.stop_time_update.departure.time])) < 1 THEN TRUE ELSE FALSE END`
False Positive Departure? | ... | String | `IF([Number of Cars]=2 AND ([Vehicle Consist]=[Prior Vehicle Consist] OR [Vehicle Consist Backwards]=[Prior Vehicle Consist]) AND ([Vehicle Consist]!=[Next Vehicle Consist] OR [Vehicle Consist Backwards]!=[Next Vehicle Consist])) THEN "False Positive" ELSEIF([Number of Cars]=2 AND ([Vehicle Consist]!=[Prior Vehicle Consist] OR [Vehicle Consist Backwards]!=[Prior Vehicle Consist]) AND ([Vehicle Consist]=[Next Vehicle Consist] OR [Vehicle Consist Backwards]=[Next Vehicle Consist])) THEN "" ELSEIF ([Number of Cars]=1 AND ([Gap to Next Terminal Departure]<5 OR ISNULL([Gap to Next Terminal Departure]))) THEN  (IF (CONTAINS([Next Vehicle Consist],[Vehicle Consist]) OR CONTAINS([Vehicle Consist After Next],[Vehicle Consist]))  THEN "False Positive"  ELSE "" END) ELSE "" END`
### Data Filters
- `trip_update.trip.revenue`=TRUE
- `trip_update.trip.schedule_relationship`!=CANCELED
- `trip_update.stop_time_update.schedule_relationship`!=SKIPPED
- `trip_update.stop_time_update.departure.time`!=NULL
- `Predictions After Departure Time`=FALSE

---

## Primary Joined Data Table
- Join the VehiclePositions and TripUpdates tables on `VehiclePositions Unique Daily Trip Identifier`=`TripUpdate Unique Daily Trip Identifier`
### Custom Calculations
Field | Description | Field Type | Query |
--- | --- | --- |  --- |
Minutes Between Predicted and Actual Departure Time | --- | Number (whole) |  `DATEDIFF('minute',[Next Departure Time],[trip_update.stop_time_update.departure.time])` |
Prediction Generated after Terminal Departure | --- | Number (whole) |  `IF (DATEDIFF('second',[TripUpdates feed_timestamp],[Next Departure Time]) <= 0) THEN TRUE ELSE FALSE END` |
Prediction Rank per Departure | ... | Number (whole) | `{PARTITION [VehiclePositions Unique Daily Trip Identifier]: {ORDERBY [TripUpdates feed_timestamp] ASC: RANK_DENSE() }}`
Advance Notice (minutes) | ... | Number (whole) | `DATEDIFF('minute',[TripUpdates feed_timestamp],[Departure Time])`
Advance Notice (minutes) per Departure | ... | Number (whole) | `ZN({ FIXED [VehiclePositions Unique Daily Trip Identifier]: MAX([Advance Notice (minutes)]) })`
Time that Departure was First Predicted | ... | String | `IF (ISNULL({ FIXED [VehiclePositions Unique Daily Trip Identifier]: MIN([TripUpdates feed_timestamp])})) THEN "No prediction was made" ELSE STR({ FIXED [VehiclePositions Unique Daily Trip Identifier]: MIN([TripUpdates feed_timestamp])}) END`
Kind | ... | String | `IF ([trip_update.stop_time_update.departure.uncertainty]='60') then "Mid-trip" elseif ([trip_update.stop_time_update.departure.uncertainty]='120') then "At Terminal" elseif ([trip_update.stop_time_update.departure.uncertainty]='360') then "Reverse Trip" elseif (ISNULL([trip_update.stop_time_update.departure.uncertainty]) and ISNULL([Earliest Prediction Generated Time per Trip])) then "No Predictions" else str([trip_update.stop_time_update.departure.uncertainty]) end`
Bin | ... | String | `IF([Advance Notice (minutes)]>=0 AND [Advance Notice (minutes)]<3) then "0-3 min" ELSEIF ([Advance Notice (minutes)]>=3 AND [Advance Notice (minutes)]<6) then "3-6 min" ELSEIF ([Advance Notice (minutes)]>=6 AND [Advance Notice (minutes)]<12) then "6-12 min" ELSEIF ([Advance Notice (minutes)]>=12 AND [Advance Notice (minutes)]<=30) then "12-30 min" elseif ([Advance Notice (minutes)]>30) then "30+ min" elseif ISNULL([Advance Notice (minutes)]) then "No Predictions" end`
Actual - Predicted | ... | Number (whole) | `DATEDIFF('second',[Departure Time],[trip_update.stop_time_update.departure.time])`
Mean Error | ... | Number (decimal) | `AVG([Actual - Predicted])`
Root Mean Squared Error | ... | Number (decimal) | `SQRT(AVG(SQUARE([Actual - Predicted])))`
Is Accurate? | ... | String | `IF([TripUpdate Unique Daily Trip Identifier]=[VehiclePositions Unique Daily Trip Identifier] and [Bin]="0-3 min") THEN (IF([Actual - Predicted]>=-60 AND [Actual - Predicted]<=60) THEN "Accurate" ELSE "Inaccurate" END) ELSEIF([TripUpdate Unique Daily Trip Identifier]=[VehiclePositions Unique Daily Trip Identifier] and [Bin]="3-6 min") THEN (IF([Actual - Predicted]>=-90 AND [Actual - Predicted]<=120) THEN "Accurate" ELSE "Inaccurate" END) ELSEIF([TripUpdate Unique Daily Trip Identifier]=[VehiclePositions Unique Daily Trip Identifier] and [Bin]="6-12 min") THEN (IF([Actual - Predicted]>=-150 AND [Actual - Predicted]<=210) THEN "Accurate" ELSE "Inaccurate" END) ELSEIF([TripUpdate Unique Daily Trip Identifier]=[VehiclePositions Unique Daily Trip Identifier] and [Bin]="12-30 min") THEN (IF([Actual - Predicted]>=-240 AND [Actual - Predicted]<=360) THEN "Accurate" ELSE "Inaccurate" END) ELSEIF([TripUpdate Unique Daily Trip Identifier]=[VehiclePositions Unique Daily Trip Identifier] and [Bin]="30+ min") THEN (IF([Actual - Predicted]>=-240 AND [Actual - Predicted]<=360) THEN "Accurate" ELSE "Inaccurate" END) end`
Prediction Available for Departure? | ... | Boolean | `IF(ISNULL([TripUpdate Unique Daily Trip Identifier]) AND NOT ISNULL([VehiclePositions Unique Daily Trip Identifier])) THEN FALSE ELSE TRUE END`
Continuous? | ... | Boolean | ...
\# Predictions | ... | Number (whole) | `COUNTD([TripUpdate Unique Prediction ID])`
\# Accurate Predictions | ... | Number (whole) | `ZN((COUNTD(IF([Is Accurate?]="Accurate") THEN ([TripUpdate Unique Prediction ID]) end)))`
% Accuracy | ... | Whole (decimal) | `[# Accurate Predictions]/[# Predictions]`
\# Departures | ... | Number (whole) | `IFNULL(COUNTD([VehiclePositions Unique Daily Trip Identifier]), 0)`
\# Predicted Departures | ... | Number (whole) | `IFNULL(countd(IF([Prediction Available for Departure?]=TRUE) THEN ([VehiclePositions Unique Daily Trip Identifier]) end),0)`
\# Departures with Continuous Coverage | ... | Number (whole) | `ZN((COUNTD(IF([Continuous?]=TRUE) THEN ([VehiclePositions Unique Daily Trip Identifier]) end)))`
% Departures with Continuous Coverage | ... | Number (decimal) | `[# Departures with Continuous Coverage]/[# Departures]`

### Data Filters
`Prediction Generated after Terminal Departure`=FALSE

---

## Secondary Joined Data Table
### Custom Calculations
Field | Description | Field Type | Query |
--- | --- | --- |  --- |
Next Prediction Rank per Departure | ... | Number (whole) | `[Prediction Rank per Departure]-1`
Next Prediction Generated Time | ... | Date & Time | Rename `[TripUpdate feed_timestamp]`
Next TripUpdate Unique Daily Trip Identifier | ... | String | Rename `[TripUpdate Unique Daily Trip Identifier]`
- Full outer join the primary joined data table and secondary joined data tables on `Prediction Rank per Departure`=`Next Prediction Rank per Departure` and `VehiclePositions Unique Daily Trip Identifier`=`Next TripUpdate Unique Daily Trip Identifier`

