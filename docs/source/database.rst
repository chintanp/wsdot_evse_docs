========
Database
========

Database Overview
=================
The database is indeed the backbone of the system. The database documentation is hosted `here`_. :numref:`evi_dss_sql` shows the various tables and relations in the database. 

.. _evi_dss_sql: 
.. figure:: _static/wsdot_evse_sql.png
    :width: 1200px
    :align: center
    :alt: ChargEval Database Schema
    :figclass: align-center

    ChargEval Database Schema

Interactive diagram

.. raw:: html

    <div style="position: relative; overflow: hidden; width: "1200"; ">
        <iframe width="1200" src='https://dbdiagram.io/embed/5dabe2ca02e6e93440f26985'> </iframe>
    </div>

Flyway
======
`Flyway`_ is used for to perform database migrations. This allows version control on database migration scripts, which is just plain sql. The `migration scripts are located here`_. The containered flyway service can be run when any database schema changes are needed.

Tables
======
The database currently has the following tables in the public schema: 

#. **analysis_record**: This is the key table in the database. Every time a user submits a request to perform analysis, a record is created in this table. The auto-increment primary key :code:`analysis_id` is used to create a one-to-many relation with several tables - :code:`evtrip_scenarios`, :code:`dest_charger`, :code:`evse_charging_session`, :code:`evse_evs_passed`, :code:`evse_power_draw`, :code:`evse_util`, :code:`ev_stranded`, :code:`ev_finished`, :code:`ev_info`, :code:`od_cd` and :code:`new_evses`. The table also has an associated trigger :code:`notify_new_order` that generates a notification using :code:`pg_notify()`, which can be used by processes listening for a notification. In the case of ChargEval, this notification is picked by the controller and an analysis request is queued. More details in the controller.

#. **wa\***: 

    * **WA_roads**: This table contains the geometry of the WA roads. The roads were made traversable for GAMA by using `clean_network()`_ and then transformed to the correct SRID as `explained here`_. Further a topology is created using the table for finding the shortest path etc.

    * **WA_roads_vertices_pgr**: This is a vertices table auto-generated by pgRouting when a topology is created. 

    * **wa_evtrips**: This table contains the output of the gravity model. For each origin and destination pair, the :code:`ret` and :code:`dep` column contain the number of returning and departing trips respectively. :code:`oevs` and :code:`devs` contain the count of EVs in origin and destination respectively, whereas :code:`ocars` and :code:`dcars` represent the count of total cars in the origin and destination respectively. 

    * **wa_gas_prices**: This table contains the average price of gas for each zip code and should be updated periodically to get the current prices. 

    * **wa_bevs**: This table contains the details about the BEVs registered in WA. This information is recieved from the WA DOL and columns like :code:`fuel_consumption` and :code:`range_fe` have been added by looking up these make and models from the fueleconomy.gov database. Column :code:`connector_code` has been added after, based on a EV manufacturer's charging standard affiliation. For example: Tesla uses Superchargers, so for all Tesla vehicles :code:`connector_code = 4`, Japanese automakers use CHAdeMO, therfore for Nissan etc. :code:`connector_code = 1`, American and German automakers use CCS, therefore for BMW etc. :code:`connector_code = 2`. 

#. **ev\***: These tables are generated by the agent-based model EVI-ABM for the EVs in the simulation - hence they use the foreign key :code:`analysis_id`.

    * **ev_finished**: For each analysis, these are the EVs that have finished their respective trips. :code:`fin_ts` represents the timestamp when the trip was finished for the vehicle with ID :code:`veh_id`. :code:`trip_distance` is the length of the shortest path between :code:`origin_zip` and :code:`destination_zip` and the :code:`distance_travelled` represents the actual distance travelled by the EV in the GAMA simulation which could include charging detours. Therefore, :math:`distance\_travelled >= trip\_distance`. Additional constraint is that combination of :code:`veh_id` and :code:`analysis_id` is unique, i.e. for analysis, a vehicle can make only one trip and hence appear in this table only once. 

    * **ev_info**: This table stores the EV info for each simulation at each timestamp. This is useful for debugging the simulation and writing to this table can be omitted once we have enough confidence in the simulation logic. This table can be deleted if the database is becoming too large. However, this will affect tabs "finished" and "stranded" in the *result viewer*. The EV info stored for the timestep :code:`simulation_ts` includes - the latitude, longitude, SOC, state, probability of charging (calculated using the charging choice decision model), to_charge boolean (probability passed to a binomial draw ultimately deciding whether the EV will charge at a charging station), and speed of the vehicle in the simulation. 

    * **ev_stranded**: This table stores the record of all the EVs stranded during the simulation, i.e. they were out of charge. This could happen, if no charging stations were available when the charge was needed. This is an indication of insufficent charging infrastructure. :code:`stranded_ts` is the timestamp when the EV was stranded. Redundant columns :code:`origin_zip` and :code:`destination_zip` added to ease the lookup, where as redundant columns :code:`stranded_lat` and :code:`stranded_lng` could be helpful to pin-point the exact location where the EV was out of charge, and useful if the table :code:`ev_info` was eliminated. 

#. **evse\***: These tables are generated by the agent-based model EVI-ABM for the EVSEs in the simulation - hence they use the foreign key :code:`analysis_id`. The agent-based simulation (aka simulation in the documentation) treats all charging stations - built as well as new alike. The field :code:`evse_id` is generated in GAMA using 

    * **evse_charging_session**: This table records all the charging sessions during a simulation. Additional constraint could be the combination of :code:`analysis_id`, :code:`veh_id` and :code:`evse_id` should be unique, i.e. a vehicle may not charge at the same charging station twice during a particular simulation.

    * **evse_evs_passed**: This table records all the EVs that passed a charging station since it was occupied. :code:`soc_val` records the SOC of the vehicle when it passed the said charging station. This is an important statistic currrently to denote EV infrastructure insufficiency and may change as a more sophisticated queueing model is implemented in GAMA.

    * **evse_util**: This table is redundant (and maybe deleted) but stores the EVSE utilization, i.e. the total energy used by a charging station durring a simulation. Additional constraint that the combination of :code:`analysis_id` and :code:`evse_id` should be unique can be enforced. 

    * **evse_power_draw**: This table stores the instantaneous power draw for all EVSEs in the simulation. 

#. **od_sp**: This is a static table and stores the shortest path lengths for all the OD pairs. 

#. **built_evse**: This table represents the charging stations that are built and operational. The charging station information is sourced from AFDC and will need to be updated regularly. 

#. **new_evses**: This table stores the information about the location, type etc. of the new charging stations being planned. 

#. **evtrip_scenarios**: This table stores the EV trips generated by the VCDM. For each analysis, based on the current infrastructure, the VCDM finds the number of EV trips between an origin and destination and determines if an EV if available in the origin zip, is likely to make a trip. If an EV is selected, then a random SOC (:code:`soc`) and a trip start time (:code:`trip_start_time`) is assigned such that the trip ends by 10pm. The :code:`veh_id` should belong to :code:`wa_bevs` and the combination of :code:`analysis_id` and :code:`veh_id` should be unique.

#. **dest_charger**: This table contains the booleans fields :code:`dc_chademo`, :code:`dc_combo` and :code:`dc_level2` to represent where a destination charger of the respective type exists at the zip code. Since, this is based on the charging infrastructure, a base value is calculated for all zip codes (and this will need to be updated periodically as the as-built condition changes), with :code:`analysis_id = -1` representing the as-built condition. For every analysis request, if the charging station acts a destination charger for a zip code, a record is added to the table with the respective :code:`analysis_id`. 

#. **zipcode_record**: This table contains details about the location of centroid of all zip codes. 

#. **user_details**: This table contains the details about the users logging onto the ChargEval. 

Besides the above tables in the public schema, the database also has an "audit" schema, that is responsible for capturing the changes on certain field. This is implemented using the `Audit Trigger`_. For example, when implemented on the :code:`analysis_record` table, the trigger captures when the status of the particular column changes. This way the time taken for a particular step (tripgen, eviabm) can be calculated and monitored over time to deduce performance trends.

Triggers 
========

1. :code:`notify_new_order()`: The trigger `notify_new_order()`_ on the table :code:`analysis_record` which notifies the listeners that a new record has been added to the table. It also converts the record to JSON and sends it along as a payload. The listener in the :Ref:`sim_man:Simulation Manager (simman)` upon receiving the notification begins the execution process. The first step is the EC2 instance launch to perform trip generation.

2. :code:`notify_trips_generated()`: The trigger `notify_trips_generated()`_ notifies its listener that the status for the analysis record row has been updated to "trips_generated". Upon this notification the :code:`simman` terminates the :code:`tripgen` EC2 instance and launches another EC2 instance to simulate the agent-based model :code:`eviabm`.

3. :code:`notify_solved()`: The trigger `notify_solved()`_ notifies that the agent-based model has solved and the :code:`simman` then terminates the said EC2 instance.


Functions
=========
The database has several functions that facilitate code re-use and modularity. 

.. _splen:

1. **sp_len(orig, dest)**: The function `sp_len(orig, dest)`_ takes the origin zip code and destination zip code as arguments and returns the shortest path length in miles between the origin and destination along the WA state road network. The shortest path is calculated using `pgr_dijkstra()`_ between the :code:`WA_roads` source vertices closest to the origin and destination zip centroids (from the :code:`zipcode_record` table)

2. **sp_od2(orig, dest)**: The function `sp_od2(orig, dest)`_ takes the origin and destination zip code and returns the geometry of the shortest path using `pgr_dijkstra()`. Of special note is the :code:`case-when-end` clause that ensures a shortest path made of segments in the correct orientation. For details and solution, refer to the `discussion`_. 


.. _notify_new_order(): https://github.com/chintanp/evi-dss/blob/0a8de620a86907342f6645e18468e37d3b5f47e0/database/migrations/V1__base_version.sql#L26
.. _sp_len(orig, dest): https://github.com/chintanp/evi-dss/blob/2aae33d0599b0fcb2fa45e807901e59132a3ec91/database/migrations/V1__base_version.sql#L41
.. _pgr_dijkstra(): http://docs.pgrouting.org/3.0/en/pgr_dijkstra.html
.. _sp_od2(orig, dest): https://github.com/chintanp/evi-dss/blob/2aae33d0599b0fcb2fa45e807901e59132a3ec91/database/migrations/V1__base_version.sql#L67
.. _discussion: https://gis.stackexchange.com/questions/334302/pgr-dijkstra-gives-wacky-routes-sometimes-with-undirected-graph
.. _clean_network(): https://gama-platform.github.io/wiki/OperatorsBC#clean_network
.. _explained here: https://gis.stackexchange.com/a/332059/18956
.. _here: https://dbdocs.io/chintanp/EVI_DSS
.. _Flyway: https://flywaydb.org/
.. _migration scripts are located here: https://github.com/chintanp/evi-dss/tree/master/database/migrations
.. _Audit Trigger: https://wiki.postgresql.org/wiki/Audit_trigger_91plus
.. _notify_trips_generated(): https://github.com/chintanp/evi-dss/blob/0a8de620a86907342f6645e18468e37d3b5f47e0/database/migrations/V4.1.1__tripgen_trigger.sql#L7
.. _notify_solved(): https://github.com/chintanp/evi-dss/blob/master/database/migrations/V6.1.2__solved_trigger.sql