# osmocom-compose
Docker compose for Osmocom with sample configuration (not guaranteed to work with new builds)

After cloning, pull osmocom submodule:
```
git submodule init
git submodule update
```

To build and run:
```
docker-compose build --build-arg USER=$USER
docker-compose up
```
