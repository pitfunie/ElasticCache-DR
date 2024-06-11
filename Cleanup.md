# Global Datastore to replicate data of Amazon ElastiCache for Redis across regions. Global Datastore instances can be 
# used in a strategy of the type Warm Standby, in order to implement disaster recovery capabilities improving the
# availability

# Remove the cluster-primary from Global Datastore. After the failover command on last step, its Role is Secondary 
# in this point.

 aws elasticache disassociate-global-replication-group \
  --global-replication-group-id ldgnf-multi-region \
  --replication-group-id cluster-primary \
  --replication-group-region us-east-1 \
  --region us-east-1
Wait until the cluster be disassociated.

# Delete cluster-primary.
 aws elasticache delete-replication-group \
  --replication-group-id cluster-primary \
  --no-retain-primary-cluster \
  --region us-east-1


# Delete Global Datastore.
 aws elasticache delete-global-replication-group \
  --global-replication-group-id ldgnf-multi-region \
  --retain-primary-replication-group \
  --region us-east-1


#Check Global DataStore status.
 aws elasticache describe-global-replication-groups \
  --show-member-info \
  --region us-east-1


# Delete cluster-secondary.
 aws elasticache delete-replication-group \
  --replication-group-id cluster-secondary \
  --no-retain-primary-cluster \
  --region us-west-1
