# Demonstration script used at the PostgreSQL User Group of 2019-10-03

## Libraries
You have to have these Python libraries installed:
- psycopg2
- tabulate
- click

## Usage
Please configure an environment variable DSN first, which psycopg2 understands.

Invoke `postgres_bloat_demo.py --help` to see the available options. Typical chain of commands:
1. prepare
2. create_bloat
3. check

You could experiment with the parameters. Defaults were used at the demonstration.

Tested with Python 3.6.8 on Ubuntu 18.04.

## Setup notes and examples

* [setup01 pip pyhton packages](setup01_pip_pyhton_packages/README.md)
* [setup02 example dsn](setup02_dsn/README.md)
* [run 01 - moderate bloat -  'Index Only Scan' a bit more expensive - 'Seq Scan' significantly more expensive ]{run01/README.md}


## Other notes

* I really like the python click package: "Click is a Python package for creating beautiful command line interfaces in a composable way with as little code as necessary." https://click.palletsprojects.com/en/7.x/ 
* A quick overview of psycopg2 https://wiki.postgresql.org/wiki/Psycopg2_Tutorial as above: "configure an environment variable DSN first, which psycopg2 understands." 
