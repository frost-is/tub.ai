# Tub.ai

Tub.ai is an  open-source tool that leverages several open source applications (RAPL, Scaphandre, Nvidia-DCGM, Node Exporter, Prometheus and Grafana) to provide researchers with more various and granular metrics. Tub.ai allows for an extraction of said metrics in a precise and concise manner using graphs and charts.

Tub.ai was first introduced in the paper "Neural network scoring for efficient computing" by Waltsburger et al., which was accepted at the International Symposium on Circuits and Systems (ISCAS) 2023

In a few months' time, the will be an entirely packaged version of tub.ai which will be easier to use and install for non technical persons. However, due to the author currently being in his last year of PhD, this will take some time.

Tub.ai can currently only be used on Linux, although, to the author's best knowledge, it is also Windows-compatible. 

# How does tub.ai work ? Part. 1 : generating the data

Tub.ai leverages several open source applications to provide a variety of metrics to system administrators and researchers.

## Initial requirements

Tub.ai is a collection of software using several technologies, which have different requirements. In order to remain as general as possible, we will nit presume the utilization of a package manager. You will need to install

- The Rust programming language, version >= 1.48. It is used by Scaphandre. Should your package manager not have a recent enough version, a static install protocol can be found [here](https://www.rust-lang.org/tools/install)
- The Go programming language, version >= 1.14. It is used by Nvidia-DCGM. Should your package manager not have a recent enough version, a static install protocol can be found [here](https://go.dev/doc/install)

## RAPL

Running Average Power Limiter (RAPL) is a protocol first introduced in Intel's 2011 Sandy Bridge architecture. By measuring the power usage of the CPU, it allowed the CPU to adjust its clock frequency so that it did not overshoot its rated Thermal Dispersion Power (TDP). After trying to create a concurrent standard, AMD also chose to adopt RAPL, which is by now available on all x86 CPUs. 

RAPL has been used in recent years as a mean of performing side-channel attacks.

There is currently no standardized equivalent for ARM or RISC-V architectures, although any such CPU having with similar capacilities could theoretically be tub.ai-compatible with some tweaking.

It should be noted that accessing RAPL measurements require a Linux kernel of version >= 5.13, as well as super-user rights

## Scaphandre

Scaphandre is an open source software written in Rust. It was created by hubblo-org and BPetit. Their repository can be found [here](https://github.com/hubblo-org/scaphandre), and their documentation [here](https://hubblo-org.github.io/scaphandre-documentation/index.html)

Scaphandre uses RAPL measurements and the CPU-time used by different PIDs to estimate the individual energy consumption of all processes running on a CPU at a given time. A lenghtier explaination can be found in Saphandre's documentation.

## Nvidia-DCGM exporter

Nvidia-DCGM (datacenter GPU manager) exporter is a software written in Go by Nvidia. It curates the data obtained by DCGM (energy consumption, memory/bandwidth/processing-power usage etc.) and exposes them on an http endpoint. More on that later. 

Installing DCGM can be done by using Nvidia's tutorial [here](https://github.com/NVIDIA/DCGM)
Installing DCGM-exporter can be done by using Nvidia's tutorial [here](https://github.com/NVIDIA/dcgm-exporter)

Do note than smaller Nvidia GPUs (MX series, Xavier series) sometimes do not come equiped with a power-probe, thus trying to obtain power readings will prove fruitless. 

## Node exporter

Node exporter is among the most complete open-source exporters. Node exporter curates a wide range of system data (including but not limited to memory usage, bandwidth usage, CPU usage, network usage etc.) and exposes them on an http endoint (once again, more on that later). If you are using AMD GPUs, Node exporter is the application that will be responsible for the export of the GPU's metrics.

Node exporter was developped by Prometheus. The installation tutorial can be found [here](https://github.com/prometheus/node_exporter)

## Brief interlude 

All the aforementioned applications serve the purpose of curating data. While the way they do it may differ, the end result is the same : they open a web server on which they display the data. After following the tutorial to the letter, commands such as `dcgm exporter`, `./node_exporter` or `scaphandre prometheus` will allow you to query using a web browser, or commands such as `wget` or `curl`. The adresse will be `http://[IP]:[port]/metrics`. If you are running tub.ai locally, you can replace [IP] by `localhost`. In the remainder of this tutorial, we will assume that you are running tub.ai locally. The default ports are the following : 

- DCGM exporter : 9400
- Node exporter : 9100 
- Scaphandre : 8080

 The output will be a yaml file looking something like this (here for a single metric among those handled by CDGM exporter)  : 

<code> HELP DCGM_FI_DEV_POWER_USAGE Power draw (in W).
 TYPE DCGM_FI_DEV_POWER_USAGE gauge
DCGM_FI_DEV_POWER_USAGE{gpu="0",UUID="GPU-2948068a-d3bd-fe5c-48c7-3b9ad325276d",device="nvidia0",modelName="NVIDIA A100-SXM4-40GB",Hostname="NvidiaDGXA100",DCGM_FI_DRIVER_VERSION="470.141.10"} 266.172000
DCGM_FI_DEV_POWER_USAGE{gpu="1",UUID="GPU-ac37caa9-f740-a02b-0ac6-5958644edaa9",device="nvidia1",modelName="NVIDIA A100-SXM4-40GB",Hostname="NvidiaDGXA100",DCGM_FI_DRIVER_VERSION="470.141.10"} 57.756000
DCGM_FI_DEV_POWER_USAGE{gpu="2",UUID="GPU-0bb0e75f-7259-f4d8-651e-0aa26a691ac9",device="nvidia2",modelName="NVIDIA A100-SXM4-40GB",Hostname="NvidiaDGXA100",DCGM_FI_DRIVER_VERSION="470.141.10"} 61.132000
DCGM_FI_DEV_POWER_USAGE{gpu="4",UUID="GPU-5f69ae21-536c-ad13-e4b2-c980eabf54b4",device="nvidia4",modelName="NVIDIA A100-SXM4-40GB",Hostname="NvidiaDGXA100",DCGM_FI_DRIVER_VERSION="470.141.10"} 63.373000</code>

These measurements' frequencies can be adjusted depending on your usecase by changing either configuration files or arguments.

So now we have the data, but we do not have a way of storing it or displaying it. The following parts will cover exactly that

# How does tub.ai work ? Part.2 : Agregating and storing the data 

## Prometheus

In order to agregate and store the data, we will need to use a timeseries database which will have to query the different exporters and store their data. Here, we will use Prometheus, but there is a variety of TSDBs available out there (Graphite, Kairos, Loki, openTSDB etc.).

At regular intervals, Prometheus will scrape the adresses which are given to it, and store the data they expose with a timestamp. 

/_!_\ By default, Prometheus only stores its data for a limited period of time so that it does not completely fill the disk. This parameter has a value of 15 days by default, but can be changed.

In order to install Prometheus, you can either download the binaries [here](https://prometheus.io/download/) - the simpler way which we recommend -, or build it from the source code |here](https://github.com/prometheus/prometheus) 

Telling prometheus which adresse to scrape can be done by editing its configuration file, `prometheus.yml`, which depending on the way you installed Prometheus, will either be in `/etc/prometheus` or in the directory in which you downloaded Prometheus. You merely have to add

<code>- job_name: "tub.ai"
    static_configs:
      - targets: ["localhost:8080", "localhost:9100", "localhost:9400"]</code>

At the end of the configuration file. You may notice that Prometheus monitors itself and exposes its data on the 9090 port. 

The default frequency at which Prometheus queries the http endpoints is `15s`, but this can be adjusted. 

# How does tub.ai work ? Part. 3 : Querying and displaying the data

 
## Directly querying timeseries with python 

Depending on the usage you intend to make of the data, you may want to query it as raw data in order to process it using you own toolchain. 

There are several ways or doing this, here we recommand using the [prometheus-pandas](https://pypi.org/project/prometheus-pandas/) python package, which will allow you to query a metric between two arbitrary dates. You can then store is as a CSV or any other data format. 

A working example of prometheus-pandas usage is provided in this repository, in the "Example - Query Prometheus Metrics" notebook.

## Displaying the data using Grafana

Prometheus provides a convenient way of storing data, but does not allow for easy visualization. This job is handled by the display engine Grafana.

Once again there is a variety of open source software designed for this job (Zabbix, LibreNMS, Icinga, Nagios etc.) but we chose to use the one we were most familiar with. Grafana is very easily interoperable with Prometheus.


The tutorial to install Grafana can be found [here](https://grafana.com/docs/grafana/latest/setup-grafana/installation/). Be sure to pick the OSS (open source software) stable version so as to avoid integration problems. 

Once Grafana has been installed, you may access it on `http://localhost:3000`, using the logins admin with password admin. 

Once this is done, you need to add Prometheus as a data source. In order to do so, click on the cogwheel down left, click "add data source", select "Prometheus" as the type, set the appropriate server URL (here it would be `localhost:9090`). If relevant, you may chose to set up the authentication, otherwise you may click "save and test". 

Now we are almost done : the only thing left to do is to add dashboards. To do that, you need to use the "dashboard" icon (the four squares). 

You may chose to create your own dashboards by clicking the "new dashboard" option. Or you may import pre-existing dashboard that fit you own needs, in this case you should click "import" and give a dashboard ID from the [Grafana dashboard library](https://grafana.com/grafana/dashboards/). 

We recommend importing 1860 (node exporter full) and 15117 (Nvidia DCGM) using Prometheus as data source. For Scaphandre, you may try to use 13845, or to create you own dashboards using the indicators available [here](https://hubblo-org.github.io/scaphandre-documentation/references/exporter-prometheus.html). 

Once you're done, congratulations, you have successfully installed tub.ai ! 

# Conclusion 

Our aim with tub.ai is to provide researchers and any interested person to easily measure the power efficiency of their algorithms. The tools we use are very common in datacenter management and Linux system administration, but see very little use outside of those communities due to the non-trivial amount of knowledge they require to be discovered and used.

This work is in progress and builds on the work of the community. We hope we can gather feedback from users and make it easier to use in the future.  
