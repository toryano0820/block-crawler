# Block Crawler

Block Crawler is a collection of tools for extracting data from a blockchain and 
storing that data in a database. While it will eventually service a number of blockchains
and databases, it is currently designed to extract NFT data from EVM blockchains. 


## Developer Setup

### Setup ENV vars

Copy `.env.local` to `.env` and fill in the `ARCHIVE_NODE_URI` entry with an actual 
archive node URI. You should be able  to leave the rest as is if you follow the defaults 
in this doc.

### Install the block-crawler app

Use pip to install the app

```bash
python -m pip install -e .
```

### Start up the local AWS services

Running the services locally saves money on the Amazon bill and reduces transaction
times. For simplicity's sake, there is a `docker-compose.yml` for just that purpose. 
Run `docker-compose up` from the project root directory, and it will launch the services
on the ports specified in the .env.local and initialize DynamoDB and S3.

### (Re-)Initialize the Database

Run the reset_db.py script to reset the database tables used by the app. You will 
need to provide the endpoint URL for the script to run. As such, it can't be used 
for non-local services.

### Running the Crawler

The Crawler has several commands. The most pertinent are crawl, seed, and listen. 
The crawl command will crawl a range of specified blocks,

Give the Crawler a spin to make sure everything is working. It requires a beginning and 
ending block to continue.

```bash
block-crawler nft crawl 1 10
```

The command above will crawl the 1st hundred blocks. It should do that quickly.

The seed command will set the last processed block in the database for the Crawler's 
tail command.


```bash
block-crawler nft seed 15_000_000
```

The command above will set the last process block number to 15,000,000. The tail command
will then start at 15,000,001 the next time it runs.

The tail command will get the last block number from the node and determine if it has 
blocks to process. It will process the blocks, sleep for a bit, and then run the entire 
process repeatedly until it is informed to stop.

## Deployment


### Creating the Config Table

To successfully run the Crawler or Tailer, you need to ensure the 
`crawler_config` table is present in the DynamoDB region they will use to store their data. 

The table name for the crawler config should be `crawler_config` prefixed by
the `TABLE_PREFIX` value you set. If your table prefix is `prefix-`, the table name
would be `prefix-crawler_config`.

The crawler config table will have a partition key named `blockchain` of type `String`.
It has no sort key or secondary indexes.


### Deploy Tailer

The Tailer is a deployment of the tail command in the `block_crawler.py` binary. It will
keep the Block Crawler database up to date by processing all blocks up to the current 
block on the connected blockchain node minus the trail blocks setting. It uses the
database to track which block it last processed to ensure that a block is not 
processed twice. **Processing a block only once is critical to ensuring accurate data.**
As such, you should **NEVER HAVE MORE THAN ONE INSTANCE OF THE TAILER FOR A BLOCKCHAIN 
RUNNING AT ANY TIME**.

#### Building a Dockerfile

The Dockerfile requires no unique settings. From the project root directory,
execute `docker build` command. An example command is below.

```bash
docker build -t chain-conductor-block-crawler-tail:latest .
```

#### Seeding the Database

Block Crawler stores information in the database to know the last processed block 
number. To do this, you must create a record in the `crawler_config` table for the 
last processed block ID of the blockchain the Tailer will be processing. Below is an 
example for setting the last block ID for the `ethereum-mainnet` blockchain to 
`123456789` in the `prefix-crawler_config` table.

The Tailer works in conjunction with the Crawler. It will always run using
the current data version of the most recent crawler run. If the Crawler has not
yet run, you will need to set the `data_version` attribute on the appropriate 
record in the `crawler_config` for the blockchain you are tailing.

```bash
aws dynamodb update-item \
--table-name=prefix-crawler_config \
--key="{\"blockchain\": {\"S\": \"ethereum-mainnet\"}}" \
--update-expression="SET last_block_id = :block_id"  \
--expression-attribute-values="{\":block_id\": {\"N\": \"123456789\"}}"
```
#### Configuring the Runtime

The runtime command will require configuration through the environment variables below:

* AWS_ACCESS_KEY_ID -- The key ID for the AWS client to connect for all AWS services
* AWS_SECRET_ACCESS_KEY -- The secret key for the AWS client to connect to all AWS 
  services
* BLOCKCHAIN -- Blockchain that will be processed
* EVM_ARCHIVE_NODE_URI -- URI to access the archive node EVM RPC HTTP server
* AWS_DYNAMODB_ENDPOINT_URL -- (optional) Override URL for connecting to Amazon 
  DynamoDB. This setting is generally only used for development.  
* AWS_S3_ENDPOINT_URL -- (optional) Override URL for connecting to Amazon S3. This 
  setting is generally only used for development.
* DYNAMODB_TIMEOUT -- (default: 5.0) Maximum time in seconds to wait for connect or 
  response from DynamoDB
* AWS_DYNAMODB_REGION -- AWS region for DynamoDB
* TABLE_PREFIX -- Table prefix for the DynamoDB table names
* AWS_S3_REGION -- AWS region for S3
* AWS_S3_METADATA_BUCKET -- S3 bucket to store metadata files
* HTTP_METADATA_TIMEOUT -- (default: 10.0) Maximum time in seconds to wait for a 
  response from an HTTP server when collecting metadata
* IPFS_NODE_URI -- URI for IPFS requests to obtain token metadata
* IPFS_METADATA_TIMEOUT -- (default: 60.0) Maximum time in seconds to wait for a 
  response from an IPFS node when collecting metadata
* ARWEAVE_NODE_URI -- URI for Arweave requests to obtain token metadata
* ARWEAVE_METADATA_TIMEOUT -- (default: 10.0) Maximum time in seconds to wait for a 
  response from an Arweave node when collecting metadata
* TRAIL_BOCKS -- (default: 1) Trail the last block by this many blocks.
* PROCESS_INTERVAL -- (default: 10.0) Minimum interval in seconds between block 
  processing actions.
* DEBUG -- (default: False) Show debug messages in the console

#### Running the Image

The image requires nothing but the environment variables to run. An example is below.

```bash
docker run \
-e BLOCKCHAIN=ethereum-mainnet \
-e AWS_ACCESS_KEY_ID=MYAWSKEYID \
... chain-conductor-block-crawler-tail:latest
```
