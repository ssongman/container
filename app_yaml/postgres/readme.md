
# server with docker


[root@okd-node04 ~]# ping okd-node04.192-168-0-229.nip.io
PING okd-node04.192-168-0-229.nip.io (192.168.0.229) 56(84) bytes of data.

[root@okd-node04 ~]#  docker run --name syj-postgres -e POSTGRES_PASSWORD=postpass -d -p5432:5432 postgres
29a2821fc9b4b2314bde84d775c38337fc88cad024f0cb1772460285965e650d





# connect 

$ psql                                    \
  --host=okd-node04.192-168-0-229.nip.io  \
  --port=5432                             \
  --username=root                         \
  --password=postpass


$ psql                                    \
  --host=okd-node04.192-168-0-229.nip.io  \
  --port=5432                             \
  --username=root


$ psql                                    \
  --host=okd-node04.192-168-0-229.nip.io  \
  --port=5432                             \
  --username=postgres

$ psql -h okd-node04.192-168-0-229.nip.io -U postgres


$ psql -h 192.168.0.229 -U postgres



$ psql -h 192.168.0.229 -p 5432 -U postgres
























