# Bundle: Power BI

This bundle contains a Microsoft Power BI example that interfaces Power BI with Geode/GemFire via the REST API. 

This article can be viewed in your browser by running the following:

```console
show_bundle geode-1-app-perf_test_powerbi-cluster-powerbi
```

## Installing Bundle

```console
install_bundle -download geode-1-app-perf_test_powerbi-cluster-powerbi
```

## Use Case

In this use case, we integrate Microsoft Power BI with Geode/GemFire using the REST API to invoke the `QueryFunction` plugin included in the `geode-addon` distribution. We join co-located regions in our queries via the `QueryFunction` plugin and perform visual analytics from Power BI.

![Power BI Data Flow Diagram](/images/powerbi.png)


## Running Cluster

The bundle installs the `powerbi` cluster which can be started as follows:

```console
# Switch into the powerbi cluster
switch_cluster powerbi

# Add a locator and two (2) members to the cluster
add_locator; add_member; add_member

# Start the cluster
start_cluster
```

## Running `perf_test_test` App

The bundle includes the `perf_test_powerbi` app with the preconfigured `etc/group-factory.properties` file for ingesting mock `Customer` and `Order` objects into the `powerbi` cluster which you started from the previous section.

Run the `test_group` command to ingest the data as follows:

```console
cd_app perf_test_powerbi; cd bin_sh

# First, built the app
./build_app

# Run test_group to ingest mock data into /nw/customers and /nw/orders
./test_group -prop ../etc/group-factory.properties -run
```

The above command ingests small sets of data as follows.

| Region        | Count |
| ------------- | ----- |
| /nw/customers | 100   |
| /nw/orders    | 1000  |

You can increase the number of entries in the `etc/group-factory.properties` file by changing the group properties.

```console
cd_app perf_test_powerbi
vi etc/group-factory.properties
```

The following properties should be changed.

```properties
# customers 
g1.totalInvocationCount=100

# orders
g3.totalInvocationCount=1000

# If you changed the customer count then you should also consider changing the
# max number of customers to include them in the generated orders. This value is
# the upper bound of the customer ID that will be assigned to orders. If you want
# all of the customers to have orders then set it to the same value as g1.totalInvocationCount.
put2.factory.customerId.max=100
```

## Loading .pbix Files

The following `.pbix` files are included in the bundle. You can load them from the Power BI Desktop.

```console
cd_app perf_test_powerbi
tree etc/powerbi/
etc/powerbi/
├── customer-orders.pbix
└── nw.pbix
```

### customer-orders.pbix

This Power BI file interfaces Geode/GemFire using the function, `QueryFunction` provided by `geode-addon`. It queries customer and order objects by executing the following OQL query:

```sql
select * from /nw/customers c, /nw/orders o where c.customerId=o.customerId
```

To join Geode/GemFire regions, the regions must be colocated. The `powerbi` cluster has been preconfigured to colocate `/nw/customers` and `/nw/orders` regions using the generic partition resolver, `IdentityKeyPartitionResolver`, provided by `geode-addon`. To properly use `IdentityKeyPartitionResolver`, the entry key must be a composite string that contains the routing key separated by the default delimiter '.'. For the `powerbi` cluster, the routing key is the second token of the entry key string. For example, the order entry key, `k0000000920.000000-0055` contains the customer ID, `000000-0055` as the routing key.

**CURL:**

```console
curl -X POST "http://localhost:7080/geode/v1/functions/addon.QueryFunction?onRegion=%2Fnw%2Forders" \
     -H "accept: application/json" -H "Content-Type: application/json" \
     -d "[ { \"@type\": \"String\",\"@value\": \"select * from /nw/customers c, /nw/orders o where c.customerId=o.customerId limit 100\"}]"
```

### nw.pbix

This Power BI file interfaces Geode/GemFire using two separate queries as follows.

```sql
select * from /nw/customers
select * from /nw/orders
```

The results of the queries are then merged into one (1) table using Power Query M.

**CURL:**

```console
# customers
curl -X GET "http://localhost:7080/geode/v1/queries/adhoc?q=select%20*%20from%20%2Fnw%2Fcustomers%20limit%20100" \
     -H "accept: application/json;charset=UTF-8"

# orders
curl -X GET "http://localhost:7080/geode/v1/queries/adhoc?q=select%20*%20from%20%2Fnw%2Forders%20limit%20100" \
     -H "accept: application/json;charset=UTF-8"
```

## Power BI Desktop

After loading the `.pbix` files, click on the *Home/Refresh* icon in the tool bar. Since the query results for both `.pbix` files are same, they should show the exact same *Freight Costs by Date* pie charts and *Customers by State* maps as shown below.

### Freight Costs by Date
![Power BI Pie Chart](/images/pbi-pie.png)

### Customers by State
![Power BI Map](/images/pbi-map.png)

## Tearing Down

```console
stop_cluster -all
```

## Conclusion

Integrating Power BI with Geode/GemFire is a trivial task using the Geode/GemFire REST API. For simple queries and small result sets, the REST API provides a quick and simple way to retrieve data in real time. However, the lack of OQL support for non-colocated data and the poor support for streaming large result sets greatly hamper its usability. The Geode/GemFire query service is not for executing complex queries and for returning large result sets. For that, a separate data extraction service is needed.
