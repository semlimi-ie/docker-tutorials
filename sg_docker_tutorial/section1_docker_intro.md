## Using docker client
- Namespacing: Isolating resources per process or group of processes
    - ex. Processes, Users, Hard drive, Hostnames, Network, Inter process communication
    - We can namesapce a process to restrict the area of a hard drive that is available
    - Kind of like redirect requests for a resource from a particular process

- Control groups: Limit amount of resources used per process
    - ex. Memory, CPU usage, HD I/O, Network bandwidth
    - Limit the amt. of memory that a process can use for example

- Limit the resource a process can use, and limit the amt. of resource it can use

&nbsp;
## How docker runs on the computer
- Docker is a Linux virtual machine and inside that all the containers are going to be created. 