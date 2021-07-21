# Robust Journey Planning

In this project we propose a _robust_ public transport route planner based for Zurich Area in Switzerland.
Given a desired arrival time, the route planner computes the fastest route between departure and arrival stops within a provided confidence tolerance expressed as interquartiles.

 ![query](figs/query.JPG)

A 7-min presentation of the project and a description of our approach and results can be found at the following [link](https://www.youtube.com/watch?v=w5wQTKwxxxA).

## Team
The project is accomplished by:
- Raphael Attias: [@raphaelattias](https://github.com/raphaelattias) (Statistics)
- Riccardo Cadei: [@riccardocadei](https://github.com/riccardocadei) (Algorithm)
- Mateo Chapatte: [@ChapatteMateo](https://github.com/ChapatteMateo)  (Front-end developer)
- Guillaume Parchet: [@Ingloryon](https://github.com/Ingloryon) (Data Engineer)


----
## Content

* [Problem Motivation](#Problem-Motivation)
* [Problem Description](#Problem-Description)
* [Running Procedure](#Running-Procedure)
* [Dataset Description](#Dataset-Description)
    - [Actual data](#Actual-data)
    - [Timetable data](#Timetable-data)
    - [Stations data](#Stations-data)
* [References](#References)

----

## Problem Motivation

Imagine you are a regular user of the public transport system, and you are checking the operator's schedule to meet your friends for a class reunion.
The choices are:

1. You could leave in 10mins, and arrive with enough time to spare for gossips before the reunion starts.

2. You could leave now on a different route and arrive just in time for the reunion.

Undoubtedly, if this is the only information available, most of us will opt for option 1.

If we now tell you that option 1 carries a fifty percent chance of missing a connection and be late for the reunion. Whereas, option 2 is almost guaranteed to take you there on time. Would you still consider option 1?

Probably not. However, most public transport applications will insist on the first option. This is because they are programmed to plan routes that offer the shortest travel times, without considering the risk factors.

[top](#Content)

----

## Problem Description

In this project we propose our _robust_ public transport route planner to improve on that, using the SBB dataset (See next section: [Dataset Description](#dataset-description)).

Given a desired arrival time, the route planner computes the fastest route between departure and arrival stops within a provided confidence tolerance expressed as interquartiles.
For instance, "what route from _A_ to _B_ is the fastest at least _Q%_ of the time if I want to arrive at _B_ before instant _T_". Note that *confidence* is a measure of a route being feasible within the travel time computed by the algorithm.

The output of the algorithm is a list of routes between _A_ and _B_ and their confidence levels. The routes are sorted from latest (fastest) to earliest (longest) departure time at _A_, they must all arrive at _B_ before _T_ with a confidence level greater than or equal to _Q_. It is also possible to visualize the routes on a map with straight lines connecting all the stops traversed by the route.

In order to answer this question we had to:

- Model the public transport infrastructure for the route planning algorithm.
- Build a predictive model using the historical arrival/departure time data, and optionally other sources of data.
- Implement a robust route planning algorithm using this predictive model.
- Test and validate the results.
- Implement a simple Jupyter-based visualization to demonstrate the method (alternative: using Jupyter dashboard such as [Voilà](https://voila.readthedocs.io/en/stable/) or [ipywidgets](https://ipywidgets.readthedocs.io/en/stable/user_guide.html)).

Solving this problem accurately can be very difficult, then we fixed the following **simplifying assumptions**:

- We only consider journeys at reasonable hours of the day, and on a typical business day.
- We allow short (total max 500m "As the Crows Flies") walking distances for transfers between two stops, and assume a walking speed of _50m/1min_ on a straight line, regardless of obstacles, human-built or natural, such as building, highways, rivers, or lakes.
- We only consider journeys that start and end on known station coordinates (train station, bus stops, etc.), never from a random location. However, walking from the departure stop to a nearby stop is allowed.
- We only consider departure and arrival stops in a 15km radius of Zürich's train station, `Zürich HB (8503000)`, (lat, lon) = `(47.378177, 8.540192)`.
- We only consider stops in the 15km radius that are reachable from Zürich HB. If needed stops may be reached via transfers through other stops outside the 15km area.
- Once a route is computed, a traveller is expected to follow the planned routes to the end, or until it fails (i.e. miss a connection).
  We haven't addressed the case where travellers are able to defer their decisions and adapt their journey "en route", as more information becomes available. This would require us to consider all alternative routes (contingency plans) in the computation of the uncertainty levels, which is more difficult to implement.
- We assumed that the timetables remain unchanged throughout the 2018 - 2020 period (i.e. schedule of May 13-17, 2019).


[top](#Content)

--- 

## Running Procedure

1. (Optional) Rederive the different datasets used from raw sources:
* Run the instructions in the notebook `PrepareTablesHDFS.ipynb`. This will create the required tables on your personnal HDFS space (on icc cluster).
* Run the cells in part I of the main notebook `ZurichJourneyPlanning.ipynb`. This will re-create the different pickles and dataframes used for the main CSA algorithm and the delays derivation. 

If you have not built the HDFS tables and run part I, you can replace in cell 2 of `ZurichJourneyPlanning.ipynb` your personal username by 'parchet' (who should already have all the tables correctly setup).

2. Find the Zurich area jounrney that fits you best !
*  Run part 2 of `ZurichJourneyPlanning.ipynb` that determines the different delays CDF aggregated by mean of transport and time of the day
*  Run part 3 to define all necessary functions for the extended CSA algorithm
*  Finally in part 4 you can play with the vizual interface to interactively find your journey based on maximal arrival time, confidence interval and locations.

3. (Optional) In the last part you can find a showcase of how our algorithm compares to sereval SBB trips

 ![query](figs/query.JPG)
 ![path1](figs/path1.JPG)
 ![path2](figs/path2.JPG)
 
 Figure 1. Example of journey display

[top](#Content)

----

## Dataset Description

For this project we used the data published on the [Open Data Platform Mobility Switzerland](<https://opentransportdata.swiss>).

We used the SBB data limited around the Zurich area, focusing only on stops within 15km of the Zurich main train station.

#### Actual data

The 2018 to 2020 data is available as a Hive table in ORC format on our HDFS system, under `/data/sbb/orc/istdaten`.

The full description of the data is available in the opentransportdata.swiss data [istdaten cookbooks](https://opentransportdata.swiss/en/cookbook/actual-data/).

- `BETRIEBSTAG`: date of the trip
- `FAHRT_BEZEICHNER`: identifies the trip
- `BETREIBER_ABK`, `BETREIBER_NAME`: operator (name will contain the full name, e.g. Schweizerische Bundesbahnen for SBB)
- `PRODUCT_ID`: type of transport, e.g. train, bus
- `LINIEN_ID`: for trains, this is the train number
- `LINIEN_TEXT`,`VERKEHRSMITTEL_TEXT`: for trains, the service type (IC, IR, RE, etc.)
- `ZUSATZFAHRT_TF`: boolean, true if this is an additional trip (not part of the regular schedule)
- `FAELLT_AUS_TF`: boolean, true if this trip failed (cancelled or not completed)
- `HALTESTELLEN_NAME`: name of the stop
- `ANKUNFTSZEIT`: arrival time at the stop according to schedule
- `AN_PROGNOSE`: actual arrival time (when `AN_PROGNOSE_STATUS` is `GESCHAETZT`)
- `AN_PROGNOSE_STATUS`: look only at lines when this is `GESCHAETZT`. This indicates that `AN_PROGNOSE` is the measured time of arrival.
- `ABFAHRTSZEIT`: departure time at the stop according to schedule
- `AB_PROGNOSE`: actual departure time (when `AN_PROGNOSE_STATUS` is `GESCHAETZT`)
- `AB_PROGNOSE_STATUS`: look only at lines when this is `GESCHAETZT`. This indicates that `AB_PROGNOSE` is the measured time of arrival.
- `DURCHFAHRT_TF`: boolean, true if the transport does not stop there

Each line of the file represents a stop and contains arrival and departure times. When the stop is the start or end of a journey, the corresponding columns will be empty (`ANKUNFTSZEIT`/`ABFAHRTSZEIT`).
In some cases, the actual times were not measured so the `AN_PROGNOSE_STATUS`/`AB_PROGNOSE_STATUS` will be empty or set to `PROGNOSE` and `AN_PROGNOSE`/`AB_PROGNOSE` will be empty.

#### Timetable data

We have copied the [timetable](https://opentransportdata.swiss/en/cookbook/gtfs/) to HDFS.

Here the timetables for the years [2018](https://opentransportdata.swiss/en/dataset/timetable-2018-gtfs), [2019](https://opentransportdata.swiss/en/dataset/timetable-2019-gtfs) and [2020](https://opentransportdata.swiss/en/dataset/timetable-2020-gtfs).
The timetables are updated weekly. 

Only GTFS format has been copied on HDFS, the full description of which is available in the opentransportdata.swiss data [timetable cookbooks](https://opentransportdata.swiss/en/cookbook/gtfs/).


We provide also a summary description of the files below. The most relevant files are marked by (+):

* stops.txt(+):

    - `STOP_ID`: unique identifier (PK) of the stop
    - `STOP_NAME`: long name of the stop
    - `STOP_LAT`: stop latitude (WGS84)
    - `STOP_LON`: stop longitude
    - `LOCATION_TYPE`:
    - `PARENT_STATION`: if the stop is one of many collocated at a same location, such as platforms at a train station

* stop_times.txt(+):

    - `TRIP_ID`: identifier (FK) of the trip, unique for the day - e.g. _1.TA.1-100-j19-1.1.H_
    - `ARRIVAL_TIME`: scheduled (local) time of arrival at the stop (same as DEPARTURE_TIME if this is the start of the journey)
    - `DEPARTURE_TIME`: scheduled (local) time of departure at the stop 
    - `STOP_ID`: stop (station) identifier (FK), from stops.txt
    - `STOP_SEQUENCE`: sequence number of the stop on this trip id, starting at 1.
    - `PICKUP_TYPE`:
    - `DROP_OFF_TYPE`:

* trips.txt:

    - `ROUTE_ID`: identifier (FK) for the route. A route is a sequence of stops. It is time independent.
    - `SERVICE_ID`: identifier (FK) of a group of trips in the calendar, and for managing exceptions (e.g. holidays, etc).
    - `TRIP_ID`: is one instance (PK) of a vehicle journey on a given route - the same route can have many trips at regular intervals; a trip may skip some of the route stops.
    - `TRIP_HEADSIGN`: displayed to passengers, most of the time this is the (short) name of the last stop.
    - `TRIP_SHORT_NAME`: internal identifier for the trip_headsign (note TRIP_HEADSIGN and TRIP_SHORT_NAME are only unique for an agency)
    - `DIRECTION_ID`: if the route is bidirectional, this field indicates the direction of the trip on the route.
    
* calendar.txt:

    - `SERVICE_ID`: identifier (PK) of a group of trips sharing a same calendar and calendar exception pattern.
    - `MONDAY`..`SUNDAY`: 0 or 1 for each day of the week, indicating occurence of the service on that day.
    - `START_DATE`: start date when weekly service id pattern is valid
    - `END_DATE`: end date after which weekly service id pattern is no longer valid
    
* routes.txt:

    - `ROUTE_ID`: identifier for the route (PK)
    - `AGENCY_ID`: identifier of the operator (FK)
    - `ROUTE_SHORT_NAME`: the short name of the route, usually a line number
    - `ROUTE_LONG_NAME`: (empty)
    - `ROUTE_DESC`: _Bus_, _Zub_, _Tram_, etc.
    - `ROUTE_TYPE`:
    
**Note:** PK=Primary Key (unique), FK=Foreign Key (refers to a Primary Key in another table)

The other files are:

* _calendar-dates.txt_ contains exceptions to the weekly patterns expressed in _calendar.txt_.
* _agency.txt_ has the details of the operators
* _transfers.txt_ contains the transfer times between stops or platforms.

Figure 1. better illustrates the above concepts relating stops, routes, trips and stop times on a real example (route _11-3-A-j19-1_, direction _0_)


 ![journeys](figs/journeys.png)
 
 _Figure 1._ Relation between stops, routes, trips and stop times. The vertical axis represents the stops along the route in the direction of travel.
             The horizontal axis represents the time of day on a non-linear scale. Solid lines connecting the stops correspond to trips.
             A trip is one instances of a vehicle journey on the route. Trips on same route do not need
             to mark all the stops on the route, resulting in trips having different stop lists for the same route.
             

#### Stations data

Consolidated list of stop locations in ORC format under `/data/sbb/orc/geostops`. The schema of this table is the same as for the `stops.txt` format described earlier.

Finally, additional stops data in [BFKOORD_GEO](https://opentransportdata.swiss/en/dataset/bhlist).
This list is older and not as complete as the stops data from the GTFS timetables. Nevertheless, it has the altitude information of the stops, which is not available from the timetable files, in case you need that.

It has the schema:

- `STATIONID`: identifier of the station/stop
- `LONGITUDE`: longitude (WGS84)
- `LATITUDE`: latitude (WGS84)
- `HEIGHT`: altitude (meters) of the stop
- `REMARK`: long name of the stop

[top](#Content) 

----

## References:

*  ["Connection Scan Algorithm", Julian Dibbelt, Thomas Pajor, Ben Strasser, Dorothea Wagner, 17-Mar-2017](https://arxiv.org/abs/1703.05997#:~:text=We%20present%20algorithm%20variants%20that,incorporation%20of%20known%20vehicle%20delays.)
* ["Intriguingly Simple and Fast Transit Routing", Julian Dibbelt, Thomas Pajor, Ben Strasser, Dorothea Wagner, Jun-2013 - page 43-54](https://link.springer.com/chapter/10.1007/978-3-642-38527-8_6)

[top](#Content)

----


