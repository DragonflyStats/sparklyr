### Connect to Spark from R. 
The sparklyr package provides a 
complete dplyr backend.
Filter and aggregate Spark datasets then bring them into R for 
analysis and visualization.
Use Spark’s distributed machine learning library from R.
Create extensions that call the full Spark API and provide 
interfaces to Spark packages.

### Installation
You can install the sparklyr package from CRAN as follows:
<pre><code>
install.packages("sparklyr")
</code></pre>
You should also install a local version of Spark for development purposes:
<pre><code>
library(sparklyr)
spark_install(version = "2.1.0")
</code></pre>
To upgrade to the latest version of sparklyr, run the following command and restart your r session:
<pre><code>
devtools::install_github("rstudio/sparklyr")
</code></pre>
If you use the RStudio IDE, you should also download the latest preview release of the IDE which includes several enhancements for interacting with Spark (see the RStudio IDE section below for more details).

###Connecting to Spark
You can connect to both local instances of Spark as well as remote Spark clusters. 
Here we’ll connect to a local instance of Spark via the spark_connect function:

<pre><code>
library(sparklyr)
sc <- spark_connect(master = "local")
</code></pre>
The returned Spark connection (sc) provides a remote dplyr data source to the Spark cluster.

For more information on connecting to remote Spark clusters see the Deployment section of the sparklyr website.
