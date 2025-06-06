# [USENIX Sec. 24] Automated Large-Scale Analysis of Cookie Notice Compliance

[Link to the paper](https://www.usenix.org/conference/usenixsecurity24/presentation/bouhoula)

## Repository structure

```shell
.
├── config               # Relevant configuration files
├── cookie_crawler       # Crawler package
│   ├── commands         # Commands to detect and explore cookie banners
│   ├── scripts          # JavaScript code needed for the crawl
│   ├── utils            # Crawl-related utility functions
│   ├── crawl_summary.py # Script to show a summary of the crawl results
│   └── run_crawler.py   # Script to run the crawler
├── classifiers          # Package for ML models
├── database             # Database package
├── docker               # Contains Docker-related configuration files
├── domains              # Cache directory for CrUX and Tranco domains
├── experiments          # Directory where the crawl data is dumped
├── shared_utils         # Contains general utility functions                       
└── run.sh               # Script to run the crawler, predictions and observatory with Docker
```

## Hardware dependencies

The artifact can run on any machine with an `x86-64` architecture CPU and the necessary software dependencies. We used a machine with a 16-core CPU (AMD 5950X), 64GB RAM, and an RTX 3080 Ti GPU.
Machines with different performance may require to adjust the `--num\_browsers` parameter, which indicates the number of parallel browsers used by the crawler. As a rule of thumb, modern CPUs can handle two browsers per physical core.

We recommend running the artifact with 50 GB of free storage on the root partition.

Note that the GPU is not mandatory to run the crawler. It is highly recommended if you would like to evaluate the ML models as it significantly speeds up training times. To use a GPU, please adapt the `docker/docker-compose.yaml` according to the instructions under the `classifier` and `ml-eval` services.

## Setup

### 1. Make sure to have [Docker](https://docs.docker.com/get-docker/) and [Docker Compose](https://docs.docker.com/compose/install/) installed.

In order to run Docker without root privileges, add your user to the docker group (`sudo usermod -a -G docker $USER`). Exit your current terminal session and start a new one for the changes to take effect.
The crawler uses the ports 5000, 5001 and 5432. Please make sure these ports are not used by other processes or adapt `docker/docker-compose.yaml` to use different ports.

### 2. [Optional] Crawl parameters

Crawl parameters can be modified in `config/experiment_config.yaml`. Alternatively, any parameter `param` can be modified using the command line argument `--<param>`. For boolean parameters use the flags `--<param>` or `--no_<param>`. For list parameters, provide each element of the list separately `--<param> val1 --<param> val2`.

For example, to set `explore_cookie_banner` to `False`, `num_browsers` to `5`, and `countries` under `domains_config` to `["de", "fr"]`, use the following arguments:

```shell
--no_explore_cookie_banner --num_browsers 5 --countries de --countries fr
```

### 3. [Optional] Database

By default, crawl results are saved to a local PostgreSQL database. Change the `DB_*` parameters in `.env` to use a different one.


### 4. [Optional] Domains

An API key is needed to download domains from the Chrome User Experience Report. Follow [this link](https://developer.chrome.com/docs/crux/api/#APIKey) to create one, and specify its path in `GOOGLE_APPLICATION_CREDENTIALS` under `.env`.

Alternatively, if you run the crawl with the default `domains_config` parameters, it will use a cached list under the `domains/` directory. You can also use a custom crawling list with the arguments `--domains_source list` and `--domains_path domains/custom_domains.txt`. Please refer to `domains/custom_domains.txt` for an example of such a list.

## Sanity check

Run

```shell
./run.sh --test
```

If this command does not encounter errors, the Docker environment was successfully built. A typical failure can be caused by insufficient storage.

This commands also starts the LibreTranslate container, which has to download several models upon creation and remains unhealthy until the download is complete. Please wait until the container is healthy before running the crawl. You can monitor this by using the `docker ps` command.

## Run

Use the following command to run the crawl:

```shell
./run.sh --crawler
```

The default parameters correspond to those used in the main crawl that we present in the paper.

You can use the ```--num_domains``` argument to sample a subset of websites to crawl instead of crawling the entire list.
For example, the following command crawls 10000 websites at random from the CRUX list we used for our crawl.
```shell
./run.sh --crawler --num_domains 10000
```

You can also change the number of parallel browsers using `--num_browsers`, and create a custom crawling list using the parameters `--domains_source` and `--domains_path`.
For example, the following command crawls the websites included in `domains/custom_domains.txt` using two parallel browsers.

```shell
./run.sh --crawler --domains_source list --domains_path domains/custom_domains.txt --num_browsers 2
```

Please check `config/experiment_config.yaml` for more details about crawl parameters and their default values.

Once the crawl finishes, get the corresponding experiment ID by running:

```shell
./run,.sh --ls
```

Then, use the following command to run predictions on the crawl results:

```shell
./run.sh --predictor --experiment_id <experiment_id>
```

Then, run the following command to display a summary of the crawl results
```shell
./run.sh --summary --experiment_id <experiment_id>
```

### Errors/warnings

* If `run.sh` fails with the error `container docker-libretranslate-cc-1 is unhealthy` simply rerunning the command should resolve the issue. When the command is executed for the first time, the LibreTranslate container has to download several models and it remains unhealthy until the download is complete.

* When running `run.sh`, please ignore the`WARN[0000] YOUR_PATH/docker-compose.yaml:version is obsolete` warning. It can be resolved by removing the first line of `docker/docker-compose.yaml`, but we keep it for backward compatibility as it does not affect functionality.

* During the crawl, please ignore the following error message that is frequently displayed.
    ```
    psycopg2.OperationalError: server closed the connection unexpectedly This probably means the server terminated abnormally before or while processing the request.
    ```

* The crawler may run into other errors if it is run with limited resources.

### Evaluate the machine learning models

Note that this will in particular perform K-fold cross-validation of the BERT models, which can be very slow without a GPU.

Run:
```shell
./run.sh --ml-eval
```

### Bring down the docker-compose services

Run:
```shell
./run.sh --down
```
