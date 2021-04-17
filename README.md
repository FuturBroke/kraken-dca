# Kraken-DCA
Kraken-DCA is a python program to automate
[Dollar Cost Averaging](https://www.investopedia.com/terms/d/dollarcostaveraging.asp) on 
[Kraken](https://kraken.com) exchange.<br>
At every launch, if no DCA pair order was already passed in the current day for the specified pair in the configuration 
file, it will create a buy limit order at current pair ask price for the specified amount in configurationn file.

Order history is saved in CSV format

The program will need a Kraken public and private API key with permissions to:
- Consult funds
- View open orders & transactions
- View closed orders & transactions
- Create and modify orders

API keys can be created from the [API page](https://www.kraken.com/u/security/api) of your Kraken account.

# Orders
The pair and the amount to DCA per day need to be specified in configuration file.

## What are the order settings ?
A buy limit taker order is created by the program at its execution, 0.26% fee are assumed.<br>
Orders are created only if no one were created during the current day for the specified pair and are immediately 
executed.<br>
Pair quote asset are used to pay Kraken fee.

## How are price, volume and fee computed ?
**Limit price**: The pair ask price at the moment of the program execution.

**Volume**: The order volume is the amount*price truncated down to the pair lot decimals, then adjusted to volume/1.0026
truncated down the pair lot decimals.<br>
By adjusting the volume, the total price of the order with fee included doesn't exceed the configuration amount.<br>

**Order price**: The order price is estimated as volume*pair_ask_price rounded to the quote asset decimals.

**Fee**: Fee are included in the specified amount by adjusting down the order volume.
0.26% taker fee are assumed and are estimated as the order_price*0.0026 round to the quote asset decimals.

Kraken documentation:
- [Kraken API documentation](https://www.kraken.com/en-us/features/api)
- [Are internal calculations made in float point or with a fixed number of decimals? Are the values always rounded?](https://support.kraken.com/hc/en-us/articles/201988998-Are-internal-calculations-made-in-float-point-or-with-a-fixed-number-of-decimals-Are-the-values-always-rounded-)
- [Assets info](https://api.kraken.com/0/public/Assets)
- [Tradable asset paird](https://api.kraken.com/0/public/AssetPairs)

## How is order history saved ?

Order history is saved in CSV format with following information per order:
- **date**: Order date.
- **pair**: Order pair, the configured DCA pair.
- **type**: Buy or sell order, *buy* in this case.
- **order_type**: Order type, *limit* in this case.
- **o_flags**: Order additional flag, *fciq* in this case to pay fee in pair quote asset.
- **pair_price**: Limit order pair price. Pair ask price at the moment of the DCA.
- **volume**: Order volume.
- **price**: Order price in pair quote asset.
- **fee**: Order fee in pair quote asset.
- **total_price**: price + fee
- **txid**: TXID of the order.
- **description**: Description of the order from Kraken.

Order history is by default saved in *orders.csv* in Kraken-DCA base directory, 
the output file can be changed through docker image execution as described below.

# Usage
## Configuration file
If you don't use docker you must edit the default *config.yaml* file.

```yaml
# Kraken's API public and private keys.
api:
  public_key: "KRAKEN_API_PUBLIC_KEY"
  private_key: "KRAKEN_API_PRIVATE_KEY"

# Pair to DCA and corresponding amount per day.
dca:
  pair: "XETHZEUR"
  amount: 20
```
- Available pairs for pair field can be found [here](https://api.kraken.com/0/public/AssetPairs) on *altname*.
- Amount of quote asset to sell to buy base asset.

More information on 
[Kraken API official documentation](https://support.kraken.com/hc/en-us/articles/360000920306-Ticker-pairs).

## Docker image
You can download the image directly from [Docker Hub](https://hub.docker.com/) using:
```sh
docker pull futurbroke/kraken-dca:latest
```
The program will be executed every hour as a cron job in a container.<br>
You must provide an empty order history CSV file at first launch. You can create one on unix system using:
```sh
touch orders.csv
```
To start the container with restart as system reboot use:
```sh
docker run -v CONFIGURATION_FILE_PATH:/app/config.yaml \
 ORDERS_FILE_PATH:/app/orders.csv \
 --name kraken-dca \
 --restart=on-failure futurbroke/kraken-dca
```
- **CONFIGURATION_FILE_PATH**: Configuration folder filepath (e.g., *~/dev/config.yaml*).
- **ORDERS_FILE_PATH**: Order history CSV filepath (e.g., *~/dev/orders.csv*).

To see container logs:
```sh
docker logs kraken-dca
```
To stop and delete the container:
```sh
docker kill kraken-dca
docker rm kraken-dca
```

## Usage without Docker
You must specify your configuration in a *config.yaml* file in the *Kraken-DCA* root folder.
### Launch Kraken-DCA
You can launch the program from the folder where you downloaded the repository folder using:
```sh
python kraken-dca
```
Or inside Kraken-DCA base directory using:
```sh
python __main__.py
```
### Automate DCA through cron
You can automate the execution by using cron on unix systems.
To execute the program every hour (it will only buy if no DCA air order was done the current day) run in a shell:
```sh
crontab -e
```
And add:
```
0 * * * * cd PROGRAM_ROOT_FOLDER && $(which python3) kraken-dca >> OUTPUT_FILE 2>&1
```
- **PROGRAM_ROOT_FOLDER**: Folder where you downloaded the repository (e.g., *~/dev*).<br>
- **OUTPUT_FILE**: Program outputs log file (e.g., *~/cron.log*).<br>

Program outputs will be available in your output file, order history in *orders.csv* in Kraken-DCA base directory.

To deactivate the cron job remove the line using again:
```sh
crontab -e
```

More crontab execution frequency options: https://crontab.guru/

# How to contribute
Thanks for your interest in contributing to the project. You can contribute freely to the project by creating an issue, 
fork it or create a pull request.
Before issuing a pull request, make sure the changes did not break any existing functionality by running unit tests in 
the Kraken-DCA base directory:
```sh
pytest
```
