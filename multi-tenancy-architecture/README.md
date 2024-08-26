# Notes from this [video](https://www.youtube.com/watch?v=IhrBgoVIoT4)

- multitenancy refers to the idea of placing multiple clients (tenants) so that they share a single resource, with the goal of maximizing profit and reducing cost.
- The multiple tenants should not be aware of each other and should be completely isolated.
- At the driver level of nvme SSDs, there're namespaces which can isolate chunks of the SSD so even the OS doesn't know there exists more storage than the namespace it's using.
- There's zoned-namespaces (in the works?) that makes mapping between one tenants data and the block on the SSD more effiecient. The video didn't delve into this however.
-
