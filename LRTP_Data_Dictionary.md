# LRTP Data Dictionary

## Primary VehiclePositions Table
### Custom Calculations
Field | Description | Field Type | Query |
--- | --- | --- |  --- |
Trip Service Date | Format `vehicle.trip.start_date` as a Date | Date | `date(DATEPARSE("yyyyMMdd", [vehicle.trip.start_date]))`
VehiclePositions Unique Daily Trip Identifier | Field to uniquely identify trip departures based on Trip Service Date and Trip ID | String | `str([Trip Service Date]) + " " + [vehicle.trip.trip_id]`
Departure Time | The time that a trip departs (the first time that the vehicle starts moving towards the following station) | Date & Time | `{ FIXED [VehiclePositions Unique Daily Trip Identifier]: MIN([VehiclePositions feed_timestamp] END)}`
Departure Stop ID | ... | String | `{fixed [VehiclePositions Unique Daily Trip Identifier]: min(if [VehiclePositions feed_timestamp] = [Departure Time] then [vehicle.stop_id] END )}`
Route | ... | String | `{fixed [VehiclePositions Unique Daily Trip Identifier]: min(if [VehiclePositions feed_timestamp] = [Departure Time] then [vehicle.trip.route_id] END )}`
Vehicle Consist | ... | String | `{fixed [VehiclePositions Unique Daily Trip Identifier]: min(if [VehiclePositions feed_timestamp] = [Departure Time] then [vehicle.vehicle.label] END )}`
Trip Terminal | ... | String | `CASE [Departure Stop ID] WHEN "70502" THEN "Union Square" WHEN "70510" THEN "Medford/Tufts" WHEN "70110" THEN "Boston College" WHEN "70236" THEN "Cleveland Circle" WHEN "70162" THEN "Riverside" WHEN "70274" THEN "Mattapan" END`
Trip Departure Rank per Terminal | ... | Number (whole) | `{PARTITION [VehiclePositions Unique Daily Trip Identifier]: {ORDERBY [Departure Time] ASC,[vehicle.trip.trip_id] ASC: RANK_DENSE() }}` |
### Data Filters
- `vehicle.trip.revenue`=TRUE
- `vehicle.trip.trip_id`!=NULL
- `vehicle.stop_id`!=NULL
- `feed_timestamp`!=NULL
- `Departure Time`!=NULL

---

## Secondary VehiclePositions Table
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
Field | Description | Field Type | Query |
--- | --- | --- |  --- |
Trip Departure Rank per Terminal After Next | ... | Number (whole) | `[Trip Departure Rank per Terminal]-2`
Vehicle Consist After Next | ... | String | Rename `[Vehicle Consist]`
Trip Terminal After Next | ... | String | Rename `[Trip Terminal]`
Trip Service Date After Next | ... | Date | Rename `[Trip Service Date]`
- Left join the primary VehiclePositions table and Tertiary VehiclePositions tables on `Trip Departure Rank per Terminal`=`Trip Departure Rank per Terminal After Next`, `Trip Service Date`=`Trip Service Date After Next`, and `Trip Terminal`=`Trip Terminal After Next`

---

## Quaternary VehiclePositions Table
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
TripUpdate Unique Prediction ID | Field to uniquely identify predictions based on the generated prediction time, the ![Uploading image.png…](), and trip ID | String | `str([TripUpdate feed_timestamp]) + " " + str([trip_update.stop_time_update.departure.time]) + " " + [trip_update.trip.trip_id]`
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

Field | Description | Field Type | Query |
--- | --- | --- |  --- |
Minutes Between Predicted and Actual Departure Time | --- | Number (whole) |  `DATEDIFF('minute',[Next Departure Time],[trip_update.stop_time_update.departure.time])` |
Prediction Generated after Terminal Departure | --- | Number (whole) |  `IF (DATEDIFF('second',[TripUpdate feed_timestamp],[Next Departure Time]) <= 0) THEN TRUE ELSE FALSE END` |
Prediction Rank per Departure | ... | Number (whole) | `{PARTITION [VehiclePositions Unique Daily Trip Identifier]: {ORDERBY [TripUpdate feed_timestamp] ASC: RANK_DENSE() }}`
### Data Filters
`Prediction Generated after Terminal Departure`=FALSE

---

## Secondary Joined Data Table
Field | Description | Field Type | Query |
--- | --- | --- |  --- |
Next Prediction Rank per Departure | ... | Number (whole) | `[Prediction Rank per Departure]-1`
Next Prediction Generated Time | ... | Date & Time | Rename `[TripUpdate feed_timestamp]`
Next TripUpdate Unique Daily Trip Identifier | ... | String | Rename `[TripUpdate Unique Daily Trip Identifier]`
- Full outer join the primary joined data table and secondary joined data tables on `Prediction Rank per Departure`=`Next Prediction Rank per Departure` and `VehiclePositions Unique Daily Trip Identifier`=`Next TripUpdate Unique Daily Trip Identifier`

