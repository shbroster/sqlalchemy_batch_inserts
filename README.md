Benchling uses SQLAlchemy and psycopg2 to talk to PostgreSQL.
To save on round-trip latency, we batch our inserts using this code.

In summary, committing 100 models in SQLAlchemy does 100 roundtrips
to the database if the model has an autoincrementing primary key.
This module improves this to 2 roundtrips without requiring any
other changes to your code.

## Usage

```python
from sqlalchemy import create_engine
from sqlalchemy.orm import sessionmaker
from sqlalchemy_batch_inserts import enable_batch_inserting

engine = create_engine("postgresql+psycopg2://postgres@localhost", use_batch_mode=True)
Session = sessionmaker(bind=engine)
session = Session()
enable_batch_inserting(session)
```

If you use [Flask-SQLALchemy](https://flask-sqlalchemy.palletsprojects.com/),

```python
from flask_sqlalchemy import SignallingSession
from sqlalchemy_batch_inserts import enable_batch_inserting

enable_batch_inserting(SignallingSession)
```

## Demo

```
docker build -t sqla_batch .
docker run --rm -v $PWD:/src --name sqla_batch sqla_batch
# Wait for it to finish spinning up
# Switch to another shell
docker exec -it sqla_batch src/demo.py no 100
docker exec -it sqla_batch src/demo.py yes 100
```

To simulate 100 * 3 inserts with 20 ms latency,
first change the connection string in demo.py from
`postgresql+psycopg2://postgres@localhost` to `postgresql+psycopg2://postgres@db`.
Then,
```
docker network create sqla_batch
docker run --rm --network sqla_batch --network-alias db --name db sqla_batch
# Switch to another shell
docker run -it -v /var/run/docker.sock:/var/run/docker.sock --network sqla_batch gaiaadm/pumba netem --duration 15m --tc-image gaiadocker/iproute2 delay --time 20 --jitter 0 db
# Switch to another shell
# This should take 100 * 3 * 20 ms = 6 seconds
docker run -it --rm -v $PWD:/src --network sqla_batch sqla_batch src/demo.py no 100
docker run -it --rm -v $PWD:/src --network sqla_batch sqla_batch src/demo.py yes 100
```

## Maintainer notes

After bumping the `version` in `setup.py` and `__version__` in `__init__.py`,

```
$ ./setup.py sdist bdist_wheel  # Generate source and py3 wheel
$ python2 setup.py bdist_wheel  # Generate py2 wheel
$ twine upload --repository-url https://test.pypi.org/legacy/ dist/*
# Check https://test.pypi.org/project/sqlalchemy-batch-inserts/
$ twine upload dist/*
```