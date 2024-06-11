# ElasticCache-DR
# --> Warm Standby: Global Datastore to replicate data of Amazon ElastiCache for Redis across regions.

# Global Datastore for Redis functionality enables you to work with fully managed cross-region replication 
# quickly, reliably and securely. Each Global Datastore is a collection of one or more cluster that replicate 
# data with each other. To use cross-region replication, you must have at least 2 clusters (Primary and Secondary, 
# or Active and Passive, respectively) configured. The primary cluster is read/write, while the cluster secondary 
# it’s just read and must be in another region.

# For the Disaster Recovery strategy, if the primary cluster shows signs of degradation, a secondary cluster can 
# be promoted to a primary cluster. The diagram below illustrates its operation:

# Create Amazon ElastiCache for Redis
# Replicate Cluster to Secondary Region
# Test Amazon ElastiCache across regions
# Promote Secondary Region to Primary


# Create AWS Cloud9 environment in Primary Region N. Virginia and the second in Region N. California
# Use and Install Redis CLI using Cloud9 Terminal
# Install Redis in N. Virginia
 # Install command-line JSON processor
 sudo yum install jq -y
 # Install dependencies
 sudo amazon-linux-extras install epel -y
 sudo yum install gcc jemalloc-devel openssl-devel tcl tcl-devel -y
 # Download the redis source code
 sudo wget http://download.redis.io/redis-stable.tar.gz
 sudo tar xvzf redis-stable.tar.gz
 cd redis-stable
 # Build Redis components
 sudo make BUILD_TLS=yes

# Use another Cloud9 Terminal in the N. California Region 
# Repeat steps 1 and 2 Secondary Region N. California
 # Install command-line JSON processor
 sudo yum install jq -y
 # Install dependencies
 sudo amazon-linux-extras install epel -y
 sudo yum install gcc jemalloc-devel openssl-devel tcl tcl-devel -y
 # Download the redis source code
 sudo wget http://download.redis.io/redis-stable.tar.gz
 sudo tar xvzf redis-stable.tar.gz
 cd redis-stable
 # Build Redis components
 sudo make BUILD_TLS=yes
Repeat steps 1 and 2 using the Secondary Region N. California



# Create an Amazon ElastiCache for Redis cluster
# Using AWS Cloud9 terminal in N. Virginia, create a new Redis Cluster (cluster-primary) in the Primary Region.
aws elasticache create-replication-group \
 --replication-group-id cluster-primary \
 --replication-group-description "DR Workshop Labs" \
 --engine redis \
 --multi-az-enabled \
 --cache-node-type cache.r6g.large \
 --num-cache-clusters 2 \
 --region us-east-1
#Please wait about 5 minutes to cluster be available.

# Check the cluster status
aws elasticache describe-replication-groups \
 --replication-group-id cluster-primary \
 --region us-east-1 |\
 jq -r .ReplicationGroups[0].Status



# Copy environment variables
# Export variables
export PRIMARY_ENDPOINT=$(aws elasticache describe-replication-groups --replication-group-id cluster-primary | jq -r '.ReplicationGroups[0].NodeGroups[0].PrimaryEndpoint.Address')
export PORT_NUMBER=$(aws elasticache describe-replication-groups --replication-group-id cluster-primary | jq -r '.ReplicationGroups[0].NodeGroups[0].PrimaryEndpoint.Port')


# Show variables values
echo $PRIMARY_ENDPOINT $PORT_NUMBER
Store keys on ElastiCache Cluster

# Connect to Cluster
src/redis-cli -h $PRIMARY_ENDPOINT -p $PORT_NUMBER


#Test the connection
 PING


# Write key-value pairs
# With the connection opened perform the REDIS command below:

 SET counter 100
 INCR counter
 INCRBY counter 10
 SET userid 5000 
 KEYS * 
 GET counter 
 GET userid 
 QUIT



# Create a Global Datastore from Regional Cluster
# Create a Global Datastore using the primary replication group.
aws elasticache create-global-replication-group \
 --global-replication-group-id-suffix multi-region \
 --primary-replication-group-id cluster-primary \
 --region us-east-1
Please wait about 5 minutes to cluster be available.



# Create new cluster in the Secondary Region N. California and add to Global Datastore
 aws elasticache create-replication-group \
 --replication-group-id cluster-secondary \
 --replication-group-description "DR Workshop Labs" \
 --global-replication-group-id ldgnf-multi-region \
 --multi-az-enabled \
 --num-cache-clusters 2 \
 --region us-west-1



# Each suffix identifies one region. The suffix “ldgnf” identifies the region N. Virginia and guarantees uniqueness 
# of the global datastore name across multiple regions. See documentation for more details.

#Check if both clusters are with Status “associated”
 aws elasticache describe-global-replication-groups \
  --global-replication-group-id ldgnf-multi-region \
  --show-member-info --region us-east-1 |\
  jq -r .GlobalReplicationGroups[0].Members
Please wait about 10-15 minutes to new cluster be associated to Global Datastore.


# Test the Secondary Region cluster
#Using the AWS Cloud9 in the secondary region N. California


# Export variables
export PRIMARY_ENDPOINT=$(aws elasticache describe-replication-groups --replication-group-id cluster-secondary --region us-west-1 | jq -r '.ReplicationGroups[0].NodeGroups[0].PrimaryEndpoint.Address')
export PORT_NUMBER=$(aws elasticache describe-replication-groups --replication-group-id cluster-secondary --region us-west-1 | jq -r '.ReplicationGroups[0].NodeGroups[0].PrimaryEndpoint.Port')


# Show variables values
echo $PRIMARY_ENDPOINT $PORT_NUMBER

#Connect to Secondary Cluster
src/redis-cli -h $PRIMARY_ENDPOINT -p $PORT_NUMBER



# Write key-value pairs
# With the connection opened perform the REDIS command below:

 KEYS *
 GET counter
 GET userid


# Try to create a new key-value pair
 SET new-key somevalue


# The message: "(error) READONLY You can’t write against a read only replica” will be shown, because it’s not permitted to
#  execute# write operation on READONLY instances. This instance ROLE is SLAVE mode.
# You can check the instance operation mode by ROLE attribute using the command below:

 INFO replication

# Promote Secondary Cluster to Primary
# Promote the cluster-secondary in N. California to primary role in the Global Datastore. Using the Cloud9 Terminal execute 
# the command below:
aws elasticache failover-global-replication-group \
 --global-replication-group-id ldgnf-multi-region \
 --primary-region us-west-1 \
 --primary-replication-group-id cluster-secondary \
--region us-east-1
The failover operation on Global Datastore has an estimated RTO of less than 1 minute.



# Using AWS Cloud9 Terminal in N. California connect to Redis again:
src/redis-cli -h $PRIMARY_ENDPOINT -p $PORT_NUMBER


# Now, try to create a new key-value pair again:
SET new-key somevalue


# Now the write operations could be performed on cluster-secondary in N. California. The same command 
# failover-global-replication-group could be used for failback operation*.
