# Monitoroing Docker Swarm Cluster with Prometheus

### Ubuntu 19.10 virtual machine

* Setup Lab environment with __Docker-Machine and Three Node Cluster__

   * Global services are services that have one running task on each cluster node, and if we add a new node to the cluster the swarm scheduler will take care of launch a new task of each global service on it.
   * Swarm provides a very simple but powerful service discovery service. It’s DNS based and when you launch a service in a overlay network each task can be discovered by any other service that lives in the same network.

1. Install Docker Machine

        curl -L https://github.com/docker/machine/releases/download/v0.12.0-rc2/docker-machine-`uname -s`-`uname -m` >/tmp/docker-machine &&
        chmod +x /tmp/docker-machine &&
        sudo cp /tmp/docker-machine /usr/local/bin/docker-machine

2. Initialize cluster
      
        for i in 1 2 3; do docker-machine create -d virtualbox --engine-opt experimental  --engine-opt metrics-addr=0.0.0.0:4999 docker-$i; done
        
        
         "--engine-opt experimental  --engine-opt metrics-addr=0.0.0.0:4999"
         
         The Docker Daemon exporter exposes Docker swarm metrics as well and its required to enable that option (experimental) and we’re going to use a specific container to expose the prometheus metrics for our Prometheus server.
         
3. Setup cluster
  * Setup master node:
        
        docker-machine ssh docker-1
        
        docker@docker-1:~$ docker swarm init --advertise-addr 192.168.99.101
        To see token
        docker@docker-1:~$ docker swarm join-token -q worker
        
  * Setup workers:
  
        docker$ docker-machine ssh docker-2

        docker@docker-2:~$ docker swarm join \
        --token SWMTKN-1-3cmw4zhu9fdep8wn162kwark8hoedtt8xw959pztgp125hlaqo-ay73hkczlbxlpmlqy3t2evdtv \
        --advertise-addr 192.168.99.102 192.168.99.101:2377

        docker-machine ssh docker-3
        
        docker@docker-3:~$ docker swarm join \
        --token SWMTKN-1-3cmw4zhu9fdep8wn162kwark8hoedtt8xw959pztgp125hlaqo-ay73hkczlbxlpmlqy3t2evdtv \
        --advertise-addr 192.168.99.103 192.168.99.101:2377

4. Build custom prometheus image with Dockerfile
        
        docker build -t localhost:5000/prometheus . 
       
5. Push it to local registry
 
  * Bring up local registry
        
         docker service create --name registry --publish 5000:5000 registry:2
  
  * Push it to registry
        
         docker push localhost:5000/prometheus

6. Deploying services to cluster
  
       docker stack deploy -c compose.yml prometheus
   
7. All metrics can are exposed over docker host ip's
        
*  Check monitored endpoints:
   
   * Prometheus
      * Prometheus service that will discover the docker metrics
        
        http://192.168.99.100:9090/targets
   
   * Node-exporter
      * Prometheus exporter for hardware and OS metrics exposed by *NIX kernels
        
        http://192.168.99.100:9100/metrics
   
   * Cadvisor
      * Ddaemon that collects, aggregates, processes, and exports information about running containers.
        
        http://192.168.99.100:8080/metrics
   
   * Socat proxy
      * Proxying to Docker engine
       
        http://192.168.99.100:4998/metrics
      
   
   * Grafana
     * http://192.168.99.100:3000/
      
 ![Setup Prometheus Datasource](https://user-images.githubusercontent.com/15921307/78150535-15c23700-7440-11ea-9a31-6d0e99eca3ff.png)

