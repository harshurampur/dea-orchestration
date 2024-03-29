# Converting ODC Metadata to Static STAC Catalogs

Static `STAC` generation is implemented using two workflows. 

1. First, the `STAC item` files are generated using a event driven pipeline, as datasets are uploaded to 
   an `S3` bucket. 

2. Second, the `STAC catalog` files are generated using batch processing
   scripts. This is to collect many child/item updates into a single update event, so that unsafe
   file updates would not occur by an otherwise asynchronous event driven implementation. 

### STAC Item Generation

This process is driven by an `SNS topic` publishing `S3 object creation` events. The 
`S3 object creation` events are pushed to an `SQS` queue which is polled by an `AWS lambda`.
The Lambda filters based on incoming file names, and generates `STAC item`. 

To install the `STAC item generation pipeline`:  


1. Deploy the `lambda` function defined in [serverless.yml](serverless.yml).

```bash
    npm install
    sls deploy
```

The above steps will deploy the lambda function in [stac.py](stac.py). 
This lambda function creates `STAC item` json files corresponding to each
dataset uploaded for a configured product.

### Catalog generation maintenance scripts

In addition to the Lambda and SQS infrastructure, the following scripts 
can be used to maintain the STAC catalog.

#### STAC Parent Update 
[stac_parent_update.py](stac_parent_update.py)

This script create/update `STAC catalog` jsons based 
on a given list of `s3 keys` corresponding to `dataset item yaml` files. This list
could be derived from an `s3 inventory list` or given on `command line`. For example,
parent catalog update corresponding to a single dataset could be done with
the following command:
    
```bash
python stac_parent_update.py -b dea-public-data \
    fractional-cover/fc/v2.2.0/ls8/x_-12/y_-12/2018/02/22/LS8_OLI_FC_3577_-12_-12_20180222125938.yaml
```

In the absence of a command line `.yaml` file list, the script derives the list
from the default inventory list (or you can specify the inventory manifest).

#### Notify to STAC SQS
[notify_to_stac_queue.py](notify_to_stac_queue.py)

This script sends messages to an SQS queue indicating modification or addition of
a list of **dataset yaml** files. 
This list can be generated from an [S3 Inventory](https://docs.aws.amazon.com/AmazonS3/latest/dev/storage-inventory.html)
or passed as command line arguments.
For example, a *STAC Item* for a dataset can be created manually, using the following command:

```bash
    python notify_to_stac_queue.py -b dea-public-data-dev \
        fractional-cover/fc/v2.2.0/ls8/x_-12/y_-12/2018/02/22/LS8_OLI_FC_3577_-12_-12_20180222125938.yaml
```
    
As with the [stac_parent_update.py](stac_parent_update.py) script, in the absence of a command line `.yaml` 
file list, the script derives the list
from the default inventory list (or you can specify the inventory manifest). 

#### Delete STAC Parent Catalogues
[delete_stac_parent_catalogs.py](delete_stac_parent_catalogs.py)
 
This script deletes all `catalog.json` objects in a bucket that start with a specified **prefix**.
This prefix typically contains a single product.



### Configuration

The configuration for everything STAC related is in [stac_config.yaml](stac_config.yaml).

This file contains cross product metadata
such as `license, contact, provider` as well as product specific details.

Each product is identified by its
*prefix* within an S3 bucket (typically [dea-public-data](https://data.dea.ga.gov.au/)).

Different versions of the same product are identified by their *prefix*. Different prefixes are
viewed as different products. The `name` of the 
product is usually the `product name` from a datacube index.

The `catalog structure` of the product must be specified. This structure is an 
ordered list of templates strings. The following rules should be followed when defining the catalog structure:

1. The last template specifies a **STAC catalog** that holds **STAC item** links. Each of
these **STAC items** are a `dataset` of the respective product and also have a 
`.yaml` file holding *ODC metadata* for the dataset.

2. The directory above that specified by the first template holds a 
**STAC Collection catalog**. If the product belongs to a *suite of products* such
as **WOfS**, there is a further top level collection catalog having links to each
of the products within that suite.

3. When defining templates, omit leading and trailing back-slashes (/). 
The following template structures have been tested:

   ```
      - x_{x}
      - x_{x}/y_{y}
   ```

   ```
      - lon_{lon}
      - lon_{lon}/lat_{lat}
   ```

   ```
      - mangrove_cover/{x}_{y}
   ```

   ```
      - L2/sentinel-2-nrt/S2MSIARD
      - L2/sentinel-2-nrt/S2MSIARD/{year:4}-{month:2}-{day:2}
   ```
   
## Manual Processing
```bash
pip install 'git+https://github.com/opendatacube/dea-proto.git#egg=odc_apps_cloud&subdirectory=apps/cloud'

# echo fractional-cover/fc/v2.2.1/ls5/x_-1/y_-11/2008/11/08/LS5_TM_FC_3577_-1_-11_20081108005928_STAC.json | jq -Rc  '{"Records": [{"s3": {"bucket": {"name": "dea-public-data"}, "object": {"key": .}}}]}' | xargs -n 1 -d '\n' aws sqs send-message --queue-url https://sqs.ap-southeast-2.amazonaws.com/538673716275/static-stac-queue --message-body

# s3-inventory-dump --prefix fractional-cover/fc/v2.2.1/ls5/ '*.yaml'

time s3-inventory-dump --prefix fractional-cover/fc/v2.2.1/ '*.yaml' > ls5_s3_yamls.txt
# Takes about 2mins 
head ls5_s3_yamls.txt | sed '/s3:\/\/dea-public-data\//!d; s///;' 


cat fc_yamls.txt | sed '/s3:\/\/dea-public-data\//!d; s///;' | xargs python notify_to_stac_queue.py -b dea-public-data -q https://sqs.ap-southeast-2.amazonaws.com/538673716275/static-stac-queue

```

# Production Info
 
```yaml
Serverless: Stack create finished...
Service Information
service: stac-catalog-generator
stage: prod
region: ap-southeast-2
stack: stac-catalog-generator-prod
resources: 8
api keys:
  None
endpoints:
  None
functions:
  stac: stac-catalog-generator-prod-stac
layers:
  None

Stack Outputs
StacLambdaFunctionQualifiedArn: arn:aws:lambda:ap-southeast-2:538673716275:function:stac-catalog-generator-prod-stac:1
QueueARN: arn:aws:sqs:ap-southeast-2:538673716275:static-stac-queue
ServerlessDeploymentBucketName: dea-lambda
QueueURL: https://sqs.ap-southeast-2.amazonaws.com/538673716275/static-stac-queue

```

## Benchmarking STAC Generation

After switching to `libyaml` based parsing, the following results were achieved for converting a set of Fractional Cover datasets to STAC. Each invocation converted 10 datasets.

| Memory Size | Duration (in ms) | Price Per 1M Invocations (in $) | 
|-------------|------------------|---------------------------------| 
| 128MB       | 625.67           | 1.46                            | 
| 256MB       | 509.72           | 2.50                            | 
| 512MB       | 173.13           | 1.67                            | 
| 1024MB      | 163.09           | 3.33                            | 
| 1536MB      | 190.67           | 5.00                            | 
| 2048MB      | 195.68           | 6.67                            | 
| 2560MB      | 174.47           | 8.34                            | 
| 3008MB      | 153.28           | 9.79                            | 

## Setting up STAC Browser

**Doesn't work yet!**

```bash

    npm install -g parcel-bundler
    git clone https://github.com/radiantearth/stac-browser.git
    cd stac-browser
    npm install
    
    # Add --public-url ./ into package.json build phase
    
    CATALOG_URL=https://data.dea.ga.gov.au/fractional-cover/fc/v2.2.1/ls5/catalog.json npm run build
    
    aws s3 cp --recursive dist s3://dea-public-data/stac-browser/
    
```