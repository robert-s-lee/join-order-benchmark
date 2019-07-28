# Simple step for Mac

- Follow the CockroachDB version of the below [instructions](https://github.com/gregrahn/join-order-benchmark).
- Start a single node CockroachDB
```
cockroach start --insecure --port=26257 --http-port=26258 --store=cockroach-data/1 --cache=256MiB --background
mkdir cockroach-data/1/extern
export IMDB=imdbload1
export EXTERN=$PWD/data/cockroach-data/1/extern
```
- Download the [datafile](http://homepages.cwi.nl/~boncz/job/imdb.tgz)
- untar into `cockroach-data/1/extern` directory

# setup DDL and indexes 

_NOTE: Schema and index are not optimized for CockroachDB performance_

```bash
cat <<EOF | cockroach sql --insecure
create database if not exists $IMDB;
use $IMDB;
\| cat schema.sql;
\| cat fkindexes.sql;
EOF
```

# export DDL for individual tables
```bash
cockroach sql --insecure --database $IMDB --format csv -e "show tables" | tail +2 | while read tbl; do
cockroach dump imdbload1 $tbl --dump-mode schema --insecure > $EXTERN/$tbl.sql
done
```

# convert datafile format
CockroachDB supports Common Format and MIME Type for Comma-Separated Values (CSV) Files format.  
`\"` has to be converted to `""` in order to be a valid [IEFT4180 CSV](https://tools.ietf.org/html/rfc4180) file.

```bash
cockroach sql --insecure --database $IMDB --format csv -e "show tables" | tail +2 | while read tbl; do
sed -i bak -e s'|\\"|""|g' $EXTERN/$tbl.csv
done
```

# run CockroachDB import command
```bash
cockroach sql --insecure --database $IMDB --format csv -e "show tables" | tail +2 | while read tbl; do
echo $tbl 
cat <<EOF | cockroach sql --insecure  --database $IMDB
drop table if exists $tbl;
IMPORT TABLE $$IMDB 
CREATE USING 'nodelocal:///$tbl.sql' 
CSV DATA ('nodelocal:///$tbl.csv');
WITH nullif = '';
EOF
done
```

# run queries
```bash
for q in `ls [0-9]*.sql | sort -g`; do
echo $q
cat $q | cockroach sql --insecure --database $IMDB --echo-sql 
done
``` 
