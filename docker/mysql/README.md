# Docker Compose: MySQL and Adminer

The Docker compose file is preconfigured to launche MySQL and Adminer containers.

1. Run MySQL and Adminer container using `docker-compose`.

```bash
cd_docker mysql
docker-compose up
```

MySQL root account is setup as follows:

| Parameter      | Value                 |
| -------------- | --------------------- |
| Adminer URL    | http://localhost:8080 |
| MySQL User     | root                  |
| MySQL Password | rootpw                |
| MySQL Port     | 3306                  |

2. To run the Pado scheduler demo, create the `nw` database using Adminer.

- Login to MySQL from Adminer URL
- Select **SQL command** from Adminer
- Execute the following:

```sql
create database nw;
```
