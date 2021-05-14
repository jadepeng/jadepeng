---
title: Docker swarm 获取service的container信息
tags: ["docker","swarm","jqpeng"]
categories: ["博客","jqpeng"]
date: 2019-09-03 08:48
---
文章作者:jqpeng
原文链接: [Docker swarm 获取service的container信息](https://www.cnblogs.com/xiaoqi/p/docker-service-container.html)

我们可以通过`docker service create`创建服务，例如：


    docker service create --name mysql mysql:latest


服务创建好后，如何来获取该service包含的容器信息呢？比如获取刚才创建的mysql服务的容器。我们可以通过docker service ps命令来获取，

## 命令行方式


    ~# docker service ps mysql
    ID                  NAME                IMAGE               NODE                DESIRED STATE       CURRENT STATE        ERROR               PORTS
    lvskmv1lkhz6        mysql.1             mysql:latest        docker86-9          Running             Running 3 days ago 


遗憾的是返回的数据不包含containerId，只有serviceId, 可以通过`docker inspect来获取service详情`


    
    ~# docker inspect lvskmv1lkhz6
    [
        {
            "ID": "lvskmv1lkhz6bvynfuxa0jqgn",
            "Version": {
                "Index": 21
            },
            "CreatedAt": "2019-08-30T08:04:18.382831966Z",
            "UpdatedAt": "2019-08-30T08:09:43.613636037Z",
            "Labels": {},
            "Spec": {
                "ContainerSpec": {
                    "Image": "mysql:latest@sha256:01cf53f2538aa805bda591d83f107c394adca8d31f98eacd3654e282dada3193",
                    "Env": [
                        "MYSQL_ROOT_PASSWORD=aimind@mysql2019\""
                    ],
                    "Isolation": "default"
                },
                "Resources": {
                    "Limits": {},
                    "Reservations": {}
                },
                "RestartPolicy": {
                    "Condition": "any",
                    "Delay": 5000000000,
                    "MaxAttempts": 0,
                    "Window": 0
                },
                "Placement": {},
                "ForceUpdate": 0
            },
            "ServiceID": "uporil7xf4rwffa0rhg1j5htw",
            "Slot": 1,
            "NodeID": "sixp62dhqe702b69pm6v8m9rh",
            "Status": {
                "Timestamp": "2019-08-30T08:09:43.554514932Z",
                "State": "running",
                "Message": "started",
                "ContainerStatus": {
                    "ContainerID": "2cf128f77797f08419f50a057973388f15753efb16134ed05370ded495d0ac08",
                    "PID": 14884,
                    "ExitCode": 0
                },
                "PortStatus": {}
            },
            "DesiredState": "running",
            "NetworksAttachments": [
                {
                    "Network": {
                        "ID": "emypqxzjggws7uicersyz6uag",
                        "Version": {
                            "Index": 12
                        },
                        "CreatedAt": "2019-08-30T08:02:57.254494392Z",
                        "UpdatedAt": "2019-08-30T08:02:57.271216394Z",
                        "Spec": {
                            "Name": "aimind-overlay",
                            "Labels": {},
                            "DriverConfiguration": {
                                "Name": "overlay"
                            },
                            "IPAMOptions": {
                                "Driver": {
                                    "Name": "default"
                                }
                            },
                            "Scope": "swarm"
                        },
                        "DriverState": {
                            "Name": "overlay",
                            "Options": {
                                "com.docker.network.driver.overlay.vxlanid_list": "4097"
                            }
                        },
                        "IPAMOptions": {
                            "Driver": {
                                "Name": "default"
                            },
                            "Configs": [
                                {
                                    "Subnet": "10.0.0.0/24",
                                    "Gateway": "10.0.0.1"
                                }
                            ]
                        }
                    },
                    "Addresses": [
                        "10.0.0.4/24"
                    ]
                }
            ]
        }
    ]
    


返回的json中，`NodeID`是所在节点ID，`Status.ContainerStatus` 是容器的状态信息，`.Status.ContainerStatus.ContainerID` 是容器ID，比如这里的是`2cf128f77797f08419f50a057973388f15753efb16134ed05370ded495d0ac08`。

拿到容器ID就能获取容器详情了，也可以获取container的统计信息：


    docker inspect 2cf128f77797f08419f50a057973388f15753efb16134ed05370ded495d0ac08
    [
        {
            "Id": "2cf128f77797f08419f50a057973388f15753efb16134ed05370ded495d0ac08",
            "Created": "2019-08-30T08:09:41.827551223Z",
            "Path": "docker-entrypoint.sh",
            "Args": [
                "mysqld"
            ],
            "State": {
                "Status": "running",
                "Running": true,
                "Paused": false,
                "Restarting": false,
                "OOMKilled": false,
                "Dead": false,
                "Pid": 14884,
                "ExitCode": 0,
                "Error": "",
                "StartedAt": "2019-08-30T08:09:43.402630785Z",
                "FinishedAt": "0001-01-01T00:00:00Z"
            },
            "Image": "sha256:62a9f311b99c24c0fde0a772abc6030bc48e5acc7d7416b8eeb72d3da1b4eb6c",
            "ResolvConfPath": "/data/docker/containers/2cf128f77797f08419f50a057973388f15753efb16134ed05370ded495d0ac08/resolv.conf",
            "HostnamePath": "/data/docker/containers/2cf128f77797f08419f50a057973388f15753efb16134ed05370ded495d0ac08/hostname",
            "HostsPath": "/data/docker/containers/2cf128f77797f08419f50a057973388f15753efb16134ed05370ded495d0ac08/hosts",
            "LogPath": "/data/docker/containers/2cf128f77797f08419f50a057973388f15753efb16134ed05370ded495d0ac08/2cf128f77797f08419f50a057973388f15753efb16134ed05370ded495d0ac08-json.log",
            "Name": "/mysql.1.lvskmv1lkhz6bvynfuxa0jqgn",
            "RestartCount": 0,
            "Driver": "overlay2",
            "Platform": "linux",
            "MountLabel": "",
            "ProcessLabel": "",
            "AppArmorProfile": "docker-default",
            "ExecIDs": null,
            "HostConfig": {
                "Binds": null,
                "ContainerIDFile": "",
                "LogConfig": {
                    "Type": "json-file",
                    "Config": {
                        "max-file": "3",
                        "max-size": "10m"
                    }
                },
                "NetworkMode": "default",
                "PortBindings": {},
                "RestartPolicy": {
                    "Name": "",
                    "MaximumRetryCount": 0
                },
                "AutoRemove": false,
                "VolumeDriver": "",
                "VolumesFrom": null,
                "CapAdd": null,
                "CapDrop": null,
                "Dns": null,
                "DnsOptions": null,
                "DnsSearch": null,
                "ExtraHosts": null,
                "GroupAdd": null,
                "IpcMode": "shareable",
                "Cgroup": "",
                "Links": null,
                "OomScoreAdj": 0,
                "PidMode": "",
                "Privileged": false,
                "PublishAllPorts": false,
                "ReadonlyRootfs": false,
                "SecurityOpt": null,
                "UTSMode": "",
                "UsernsMode": "",
                "ShmSize": 67108864,
                "Runtime": "runc",
                "ConsoleSize": [
                    0,
                    0
                ],
                "Isolation": "default",
                "CpuShares": 0,
                "Memory": 0,
                "NanoCpus": 0,
                "CgroupParent": "",
                "BlkioWeight": 0,
                "BlkioWeightDevice": null,
                "BlkioDeviceReadBps": null,
                "BlkioDeviceWriteBps": null,
                "BlkioDeviceReadIOps": null,
                "BlkioDeviceWriteIOps": null,
                "CpuPeriod": 0,
                "CpuQuota": 0,
                "CpuRealtimePeriod": 0,
                "CpuRealtimeRuntime": 0,
                "CpusetCpus": "",
                "CpusetMems": "",
                "Devices": null,
                "DeviceCgroupRules": null,
                "DiskQuota": 0,
                "KernelMemory": 0,
                "MemoryReservation": 0,
                "MemorySwap": 0,
                "MemorySwappiness": null,
                "OomKillDisable": false,
                "PidsLimit": 0,
                "Ulimits": null,
                "CpuCount": 0,
                "CpuPercent": 0,
                "IOMaximumIOps": 0,
                "IOMaximumBandwidth": 0,
                "MaskedPaths": [
                    "/proc/acpi",
                    "/proc/kcore",
                    "/proc/keys",
                    "/proc/latency_stats",
                    "/proc/timer_list",
                    "/proc/timer_stats",
                    "/proc/sched_debug",
                    "/proc/scsi",
                    "/sys/firmware"
                ],
                "ReadonlyPaths": [
                    "/proc/asound",
                    "/proc/bus",
                    "/proc/fs",
                    "/proc/irq",
                    "/proc/sys",
                    "/proc/sysrq-trigger"
                ]
            },
            "GraphDriver": {
                "Data": {
                    "LowerDir": "/data/docker/overlay2/f0184a2c979eef7a135726a49f5651e16b568ecfd47606e20e504e28ea311f25-init/diff:/data/docker/overlay2/644c4c905af78d3320559b9f388631151dcf5c19ab8f2c91999d4d59c8409784/diff:/data/docker/overlay2/7ed834798bd5eeef1b75d012a27bb01cd8a0a5e71048db72a8743980481bb74b/diff:/data/docker/overlay2/56e3eac1c86a9ae29b3251025824f93b78e43151a36eb973407feb1075d8db1c/diff:/data/docker/overlay2/40161cfa334a118eaa09c04dc7d864d00e3544f77e6979584298478f68566bc5/diff:/data/docker/overlay2/e884a3df3e827368a468a4afc8850de4fa6336a78ca9a922406237e3ab75a97e/diff:/data/docker/overlay2/a04e8776674f902eaa0e15467ad0678f03baf2a1b8a568b034ad4b4c1ddb1a23/diff:/data/docker/overlay2/7745739e901232d6b702b599844157583d02a34fa4aca10c888e0e9c44075433/diff:/data/docker/overlay2/f423b8f55475ec902cea1ea5c54897ed6a24da3cc0acd64a79e022e887d83e77/diff:/data/docker/overlay2/231e63e7fbb5084facc93c89ed23d366d915f9a2edd4f85735df5d45bc87cafa/diff:/data/docker/overlay2/c11047327e6f47e49d1abee4df8acbaba51ac6b92e59801ac613331c5bad3bc1/diff:/data/docker/overlay2/f893602043c1b5ad9d2839ec0ab8f17da7e0eaf073788f6c3d35138dfe6c06b8/diff:/data/docker/overlay2/3443517fc9e882df67d9730a9aa7530dc3c541b6872aaf05290c5e7ec588e0fb/diff",
                    "MergedDir": "/data/docker/overlay2/f0184a2c979eef7a135726a49f5651e16b568ecfd47606e20e504e28ea311f25/merged",
                    "UpperDir": "/data/docker/overlay2/f0184a2c979eef7a135726a49f5651e16b568ecfd47606e20e504e28ea311f25/diff",
                    "WorkDir": "/data/docker/overlay2/f0184a2c979eef7a135726a49f5651e16b568ecfd47606e20e504e28ea311f25/work"
                },
                "Name": "overlay2"
            },
            "Mounts": [
                {
                    "Type": "volume",
                    "Name": "c2128d05001b8fec1712807f381e2c72d42ce8a83ae97f6b038f51c0d48446f1",
                    "Source": "/data/docker/volumes/c2128d05001b8fec1712807f381e2c72d42ce8a83ae97f6b038f51c0d48446f1/_data",
                    "Destination": "/var/lib/mysql",
                    "Driver": "local",
                    "Mode": "",
                    "RW": true,
                    "Propagation": ""
                }
            ],
            "Config": {
                "Hostname": "2cf128f77797",
                "Domainname": "",
                "User": "",
                "AttachStdin": false,
                "AttachStdout": false,
                "AttachStderr": false,
                "ExposedPorts": {
                    "3306/tcp": {},
                    "33060/tcp": {}
                },
                "Tty": false,
                "OpenStdin": false,
                "StdinOnce": false,
                "Env": [
                    "MYSQL_ROOT_PASSWORD=aimind@mysql2019\"",
                    "PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin",
                    "GOSU_VERSION=1.7",
                    "MYSQL_MAJOR=8.0",
                    "MYSQL_VERSION=8.0.17-1debian9"
                ],
                "Cmd": [
                    "mysqld"
                ],
                "ArgsEscaped": true,
                "Image": "mysql:latest@sha256:01cf53f2538aa805bda591d83f107c394adca8d31f98eacd3654e282dada3193",
                "Volumes": {
                    "/var/lib/mysql": {}
                },
                "WorkingDir": "",
                "Entrypoint": [
                    "docker-entrypoint.sh"
                ],
                "OnBuild": null,
                "Labels": {
                    "com.docker.swarm.node.id": "sixp62dhqe702b69pm6v8m9rh",
                    "com.docker.swarm.service.id": "uporil7xf4rwffa0rhg1j5htw",
                    "com.docker.swarm.service.name": "mysql",
                    "com.docker.swarm.task": "",
                    "com.docker.swarm.task.id": "lvskmv1lkhz6bvynfuxa0jqgn",
                    "com.docker.swarm.task.name": "mysql.1.lvskmv1lkhz6bvynfuxa0jqgn"
                }
            },
            "NetworkSettings": {
                "Bridge": "",
                "SandboxID": "459ab4b83580513da251182d08dc217d0079613d10952df00ffcca6e2537958b",
                "HairpinMode": false,
                "LinkLocalIPv6Address": "",
                "LinkLocalIPv6PrefixLen": 0,
                "Ports": {
                    "3306/tcp": null,
                    "33060/tcp": null
                },
                "SandboxKey": "/var/run/docker/netns/459ab4b83580",
                "SecondaryIPAddresses": null,
                "SecondaryIPv6Addresses": null,
                "EndpointID": "",
                "Gateway": "",
                "GlobalIPv6Address": "",
                "GlobalIPv6PrefixLen": 0,
                "IPAddress": "",
                "IPPrefixLen": 0,
                "IPv6Gateway": "",
                "MacAddress": "",
                "Networks": {
                    "aimind-overlay": {
                        "IPAMConfig": {
                            "IPv4Address": "10.0.0.4"
                        },
                        "Links": null,
                        "Aliases": [
                            "2cf128f77797"
                        ],
                        "NetworkID": "emypqxzjggws7uicersyz6uag",
                        "EndpointID": "56a78b2527a6dcf83fd3dc2794c514aaa325457d9c8a21bd236d3ea3c22c8fa9",
                        "Gateway": "",
                        "IPAddress": "10.0.0.4",
                        "IPPrefixLen": 24,
                        "IPv6Gateway": "",
                        "GlobalIPv6Address": "",
                        "GlobalIPv6PrefixLen": 0,
                        "MacAddress": "02:42:0a:00:00:04",
                        "DriverOpts": null
                    }
                }
            }
        }
    ]
    


然后就可以通过stats来获取资源占用情况：


    ~#docker stats 2cf128f77797f08419f50a057973388f15753efb16134ed05370ded495d0ac08 --all --no-stream 
    CONTAINER ID        NAME                                CPU %               MEM USAGE / LIMIT     MEM %               NET I/O             BLOCK I/O           PIDS
    2cf128f77797        mysql.1.lvskmv1lkhz6bvynfuxa0jqgn   0.33%               374.4MiB / 188.8GiB   0.19%               230kB / 0B          8.19kB / 1.26GB     38


## coding方式

除了命令行，我们还可以通过docker api来获取，可以参见 [docker-java Docker的java API](https://www.cnblogs.com/xiaoqi/p/docker-java.html)

### 获取containerID


    System.out.println(client.listTasksCmd().withNameFilter("mysql").exec());


结果：


    [class Task {
        ID: lvskmv1lkhz6bvynfuxa0jqgn
        version: 21
        createdAt: 2019-08-30T08:04:18.382831966Z
        updatedAt: 2019-08-30T08:09:43.613636037Z
        name: null
        labels: {}
        spec: TaskSpec[containerSpec=ContainerSpec[image=mysql:latest@sha256:01cf53f2538aa805bda591d83f107c394adca8d31f98eacd3654e282dada3193,labels=<null>,command=<null>,args=<null>,env=[MYSQL_ROOT_PASSWORD=aimind@mysql2019"],dir=<null>,user=<null>,groups=<null>,tty=<null>,mounts=<null>,duration=<null>,stopGracePeriod=<null>,dnsConfig=<null>,openStdin=<null>,readOnly=<null>,hosts=<null>,hostname=<null>,secrets=<null>,healthCheck=<null>,stopSignal=<null>,privileges=<null>,configs=<null>],resources=ResourceRequirements[limits=ResourceSpecs[memoryBytes=<null>,nanoCPUs=<null>],reservations=ResourceSpecs[memoryBytes=<null>,nanoCPUs=<null>]],restartPolicy=ServiceRestartPolicy[condition=ANY,delay=5000000000,maxAttempts=0,window=0],placement=ServicePlacement[constraints=<null>,platforms=<null>],logDriver=<null>,forceUpdate=0,networks=<null>,runtime=<null>]
        serviceId: uporil7xf4rwffa0rhg1j5htw
        slot: 1
        nodeId: sixp62dhqe702b69pm6v8m9rh
        assignedGenericResources: null
        status: TaskStatus[timestamp=2019-08-30T08:09:43.554514932Z,state=running,message=started,err=<null>,containerStatus=TaskStatusContainerStatus[containerID=2cf128f77797f08419f50a057973388f15753efb16134ed05370ded495d0ac08,pid=14884,exitCode=0]]
        desiredState: running
    }]


可以看到containerID：`2cf128f77797f08419f50a057973388f15753efb16134ed05370ded495d0ac08` 和命令行一直。

然后获取容器详情：


     System.out.println(client.inspectContainerCmd("2cf128f77797f08419f50a057973388f15753efb16134ed05370ded495d0ac08").exec());


获取容器统计信息：


            System.out.println(client.statsCmd("2cf128f77797f08419f50a057973388f15753efb16134ed05370ded495d0ac08").exec(new InvocationBuilder.AsyncResultCallback<>()).awaitResult());
    


对应的结果：


    InspectContainerResponse[args={mysqld},config=com.github.dockerjava.api.model.ContainerConfig@3e15bb06[attachStderr=false,attachStdin=false,attachStdout=false,cmd={mysqld},domainName=,entrypoint={docker-entrypoint.sh},env={MYSQL_ROOT_PASSWORD=aimind@mysql2019",PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin,GOSU_VERSION=1.7,MYSQL_MAJOR=8.0,MYSQL_VERSION=8.0.17-1debian9},exposedPorts=com.github.dockerjava.api.model.ExposedPorts@6778aea6,hostName=2cf128f77797,image=mysql:latest@sha256:01cf53f2538aa805bda591d83f107c394adca8d31f98eacd3654e282dada3193,labels={com.docker.swarm.node.id=sixp62dhqe702b69pm6v8m9rh, com.docker.swarm.service.id=uporil7xf4rwffa0rhg1j5htw, com.docker.swarm.service.name=mysql, com.docker.swarm.task=, com.docker.swarm.task.id=lvskmv1lkhz6bvynfuxa0jqgn, com.docker.swarm.task.name=mysql.1.lvskmv1lkhz6bvynfuxa0jqgn},macAddress=<null>,networkDisabled=<null>,onBuild=<null>,stdinOpen=false,portSpecs=<null>,stdInOnce=false,tty=false,user=,volumes={/var/lib/mysql={}},workingDir=,healthCheck=<null>],created=2019-08-30T08:09:41.827551223Z,driver=overlay2,execDriver=<null>,hostConfig=com.github.dockerjava.api.model.HostConfig@5853495b[binds=<null>,blkioWeight=0,blkioWeightDevice=<null>,blkioDeviceReadBps=<null>,blkioDeviceWriteBps=<null>,blkioDeviceReadIOps=<null>,blkioDeviceWriteIOps=<null>,memorySwappiness=<null>,nanoCPUs=<null>,capAdd=<null>,capDrop=<null>,containerIDFile=,cpuPeriod=0,cpuRealtimePeriod=0,cpuRealtimeRuntime=0,cpuShares=0,cpuQuota=0,cpusetCpus=,cpusetMems=,devices=<null>,deviceCgroupRules=<null>,diskQuota=0,dns=<null>,dnsOptions=<null>,dnsSearch=<null>,extraHosts=<null>,groupAdd=<null>,ipcMode=shareable,cgroup=,links=<null>,logConfig=com.github.dockerjava.api.model.LogConfig@524a2ffb,lxcConf=<null>,memory=0,memorySwap=0,memoryReservation=0,kernelMemory=0,networkMode=default,oomKillDisable=false,init=<null>,autoRemove=false,oomScoreAdj=0,portBindings={},privileged=false,publishAllPorts=false,readonlyRootfs=false,restartPolicy=no,ulimits=<null>,cpuCount=0,cpuPercent=0,ioMaximumIOps=0,ioMaximumBandwidth=0,volumesFrom=<null>,mounts=<null>,pidMode=,isolation=default,securityOpts=<null>,storageOpt=<null>,cgroupParent=,volumeDriver=,shmSize=67108864,pidsLimit=0,runtime=runc,tmpFs=<null>,utSMode=,usernsMode=,sysctls=<null>,consoleSize=[0, 0]],hostnamePath=/data/docker/containers/2cf128f77797f08419f50a057973388f15753efb16134ed05370ded495d0ac08/hostname,hostsPath=/data/docker/containers/2cf128f77797f08419f50a057973388f15753efb16134ed05370ded495d0ac08/hosts,logPath=/data/docker/containers/2cf128f77797f08419f50a057973388f15753efb16134ed05370ded495d0ac08/2cf128f77797f08419f50a057973388f15753efb16134ed05370ded495d0ac08-json.log,id=2cf128f77797f08419f50a057973388f15753efb16134ed05370ded495d0ac08,sizeRootFs=<null>,imageId=sha256:62a9f311b99c24c0fde0a772abc6030bc48e5acc7d7416b8eeb72d3da1b4eb6c,mountLabel=,name=/mysql.1.lvskmv1lkhz6bvynfuxa0jqgn,restartCount=0,networkSettings=com.github.dockerjava.api.model.NetworkSettings@7173ae5b[bridge=,sandboxId=459ab4b83580513da251182d08dc217d0079613d10952df00ffcca6e2537958b,hairpinMode=false,linkLocalIPv6Address=,linkLocalIPv6PrefixLen=0,ports={3306/tcp=null, 33060/tcp=null},sandboxKey=/var/run/docker/netns/459ab4b83580,secondaryIPAddresses=<null>,secondaryIPv6Addresses=<null>,endpointID=,gateway=,portMapping=<null>,globalIPv6Address=,globalIPv6PrefixLen=0,ipAddress=,ipPrefixLen=0,ipV6Gateway=,macAddress=,networks={aimind-overlay=com.github.dockerjava.api.model.ContainerNetwork@53a9fcfd[ipamConfig=com.github.dockerjava.api.model.ContainerNetwork$Ipam@21f459fc,links=<null>,aliases=[2cf128f77797],networkID=emypqxzjggws7uicersyz6uag,endpointId=56a78b2527a6dcf83fd3dc2794c514aaa325457d9c8a21bd236d3ea3c22c8fa9,gateway=,ipAddress=10.0.0.4,ipPrefixLen=24,ipV6Gateway=,globalIPv6Address=,globalIPv6PrefixLen=0,macAddress=02:42:0a:00:00:04]}],path=docker-entrypoint.sh,processLabel=,resolvConfPath=/data/docker/containers/2cf128f77797f08419f50a057973388f15753efb16134ed05370ded495d0ac08/resolv.conf,execIds=<null>,state=com.github.dockerjava.api.command.InspectContainerResponse$ContainerState@4d192aef[status=running,running=true,paused=false,restarting=false,oomKilled=false,dead=false,pid=14884,exitCode=0,error=,startedAt=2019-08-30T08:09:43.402630785Z,finishedAt=0001-01-01T00:00:00Z,health=<null>],volumes=<null>,volumesRW=<null>,node=<null>,mounts=[com.github.dockerjava.api.command.InspectContainerResponse$Mount@1416cf9f[name=c2128d05001b8fec1712807f381e2c72d42ce8a83ae97f6b038f51c0d48446f1,source=/data/docker/volumes/c2128d05001b8fec1712807f381e2c72d42ce8a83ae97f6b038f51c0d48446f1/_data,destination=/var/lib/mysql,driver=local,mode=,rw=true]],graphDriver=com.github.dockerjava.api.command.GraphDriver@84487f4[name=overlay2,data=com.github.dockerjava.api.command.GraphData@bfc14b9[rootDir=<null>,deviceId=<null>,deviceName=<null>,deviceSize=<null>,dir=<null>]],platform=linux]
    Disconnected from the target VM, address: '127.0.0.1:60730', transport: 'socket'
    com.github.dockerjava.api.model.Statistics@55a88417[read=2019-09-02T12:20:14.534216408Z,networks={eth0=com.github.dockerjava.api.model.StatisticNetworksConfig@18acfe88[rxBytes=0,rxDropped=0,rxErrors=0,rxPackets=0,txBytes=0,txDropped=0,txErrors=0,txPackets=0], eth1=com.github.dockerjava.api.model.StatisticNetworksConfig@8a2a6a[rxBytes=197752,rxDropped=0,rxErrors=0,rxPackets=836,txBytes=0,txDropped=0,txErrors=0,txPackets=0]},network=<null>,memoryStats=com.github.dockerjava.api.model.MemoryStatsConfig@772861aa,blkioStats=BlkioStatsConfig[ioServiceBytesRecursive=[BlkioStatEntry[major=8,minor=0,op=Read,value=8192], BlkioStatEntry[major=8,minor=0,op=Write,value=1259921408], BlkioStatEntry[major=8,minor=0,op=Sync,value=1258987520], BlkioStatEntry[major=8,minor=0,op=Async,value=942080], BlkioStatEntry[major=8,minor=0,op=Total,value=1259929600]],ioServicedRecursive=[BlkioStatEntry[major=8,minor=0,op=Read,value=2], BlkioStatEntry[major=8,minor=0,op=Write,value=4066], BlkioStatEntry[major=8,minor=0,op=Sync,value=4009], BlkioStatEntry[major=8,minor=0,op=Async,value=59], BlkioStatEntry[major=8,minor=0,op=Total,value=4068]],ioQueueRecursive=[],ioServiceTimeRecursive=[],ioWaitTimeRecursive=[],ioMergedRecursive=[],ioTimeRecursive=[],sectorsRecursive=[]],cpuStats=com.github.dockerjava.api.model.CpuStatsConfig@4cb40e3b,preCpuStats=com.github.dockerjava.api.model.CpuStatsConfig@41b1f51e,pidsStats=com.github.dockerjava.api.model.PidsStatsConfig@3a543f31]


* * *


> 作者：Jadepeng  
>  出处：jqpeng的技术记事本--[http://www.cnblogs.com/xiaoqi](http://www.cnblogs.com/xiaoqi)  
>  您的支持是对博主最大的鼓励，感谢您的认真阅读。  
>  本文版权归作者所有，欢迎转载，但未经作者同意必须保留此段声明，且在文章页面明显位置给出原文连接，否则保留追究法律责任的权利。


