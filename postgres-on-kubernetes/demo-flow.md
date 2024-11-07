1. Install operator.
   
   ```
   kubectl apply -f \
    https://github.com/cloudnative-pg/cloudnative-pg/releases/download/v1.22.0/cnpg-1.22.0.yaml
   ```

   Install also a `cnpg` plugin for `kubectl`, using `krew`, the `kubectl`
   plugin manager.

   ```
   kubectl krew install cnpg
   ```
   
   Note what is installed (the controller, CRDs).

   ```
   k get all -n cnpg-system
   ```

2. Create a minimal cluster.
   
   Show how little is needed to set up a functioning cluster.

   ```
   k apply -f demo/cluster.minimal.yml
   ```

   Note what is installed (pods, services, secrets).

   ```
   k get all -n cnpg-system
   ```

   Show that it's accessible and replication works. Here we probably need a
   client. 

   ```
   kubectl cnpg status demo
   ```

   Also, use `kubectl cnpg psql demo` for the primary node and `kubectl cnpg psql demo --replica`
   for one of the replicas.

3. Show failover.

   Prepare the pgbench data.

   ```
   kubectl cnpg pgbench pg15 -- -i -s 100
   ```

   Create a ConfigMap with the pgbench wrapper script.

   ```
   kubectl apply -f demo/configmap.scripts.yml
   ```

   Run `kubectl cnpg status demo` to determine which pod is the primary and
   on what node it is running.

   Launch the job that runs the wrapper for two minutes.

   ```
   kubectl apply -f demo/job.pgbench-with-wrapper.yml
   ```
   
   Simulate the node failure by stopping the primary node's container (which we 
   identified earlier).

   ```
   podman stop cnpg-worker3
   ```
   
   Wait until the pgbench job finishes (`kubectl get jobs`), then display the
   job's pod log, which will contain the failover latency.

   ```
   kubectl logs demo-pgbench-failover-5zb7n
   ```

   Note: the cluster pod will be migrated to another node after the `TolerationSeconds`
   interval, which is 300 by default.

   Start the worker container: `podman start cnpg-worker3`

   Check the cluster status after a minute or two: `kubectl cnpg status demo`.

   Now try switchover while running the same pgbench test. Start the job, then
   use `kubectl cnpg promote demo-3`.

   Wait for the job completion, then display the job pod log.

3. Show cluster initialisation from an existing database.

   ## Create ConfigMaps with Northwind scripts

   ```
   kubectl create configmap northwind --from-file=northwind_ddl.sql=demo/northwind_ddl.sql --from-file=northwind_data.sql=demo/northwind_data.sql
   ```

   ## Create source cluster

   Create a copy of the minimal cluster and add `imageName: ghcr.io/cloudnative-pg/postgresql:11`
   and set up a standalone instance (no replication).

   Use post-init scripts to create the sample contents.  The
   scripts are stored in a ConfigMap.


   ```
   projectedVolumeTemplate:
      sources:
      -  configMap:
            name: northwind
            items:
            -  key: northwind_ddl.sql
               path: northwind_ddl.sql
            -  key: northwind_data.sql
               path: northwind_data.sql
   ```

   ```
   bootstrap:
      initdb:
         database: nw
         postInitApplicationSQLRefs:
            configMapRefs:
            - name: northwind
              key: northwind_ddl.sql
            - name: northwind
              key: northwind_data.sql
   ```

   Rune `kubectl cnpg status` to ensure the new cluster is available.

   Run `kubectl cnpg psql pg11 -- -d nw` to see the schema exists.

   ## Init

   Create a copy of the minimal cluster, add an external cluster definition

   ```
   externalClusters:
   -  name: pg11
      connectionParameters:
         host: pg11-r
         user: nw
         dbname: nw
      password:
         name: pg11-app
         key: password
   ```
   
   and a bootstrap section specifying the import source.

   ```
   bootstrap:
      initdb:
         database: nw
         import:
            type: microservice
            databases:
               - nw
            source:
               externalCluster: pg11
   ```

   Create the cluster, connect to it and check that data are there.

4. Show monitoring.

   First, with `kubectl cnpg`. 

   Deploy Prometheus stack 
   ```
   helm repo add prometheus-community \
   https://prometheus-community.github.io/helm-charts

   helm upgrade --install \
   -f resources/prom-config.yml \
   prom \
   prometheus-community/kube-prometheus-stack
   ```

   Create pod monitor

   ```
   kubectl edit cluster demo
   ```

   Enable access to Prometheus

   ```
   kubectl port-forward svc/prom-kube-prometheus-stack-prometheus 9090
   ```

   Create Grafana dashboard

