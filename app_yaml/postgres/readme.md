
# server with docker

docker run --name syj-postgres -e POSTGRES_PASSWORD=postpass -d -p5432:5432 postgres



# connect 

$ psql                                    \
  --host=okd-node04.192-168-0-229.nip.io  \
  --port=5432                             \
  --username=root                         \
  --password=postpass

