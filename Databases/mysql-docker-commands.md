1. run a command in container without entering the container:

```sql
docker exec -i ecomm-mysql mysql -uroot -ppassword <<< "CREATE DATABASE ecomm;"
```

"ecomm-mysql"  ==> database name 
"root"  ==> database username
"password" ==> database password
"CREATE DATABASE ecomm;" ==> the command which we want to execute in the mysql container


روش دوم برای اینکه بتونیم به برنامه ما اس کیو ال ورودی پاس دهیم:

```sql
echo "CREATE DATABASE ecomm;" | docker exec -i ecomm-mysql mysql -uroot -ppassword
```


روش سوم:

```sql
docker exec ecomm-mysql \
  mysql -uroot -ppassword \
  -e "CREATE DATABASE ecomm;"
```





