### max threads on linux

linux restricts the number of threads that a user can run by default
* ubuntu 16.04: 2000 ports
docker gets around this to a degree
to run directly without running as root some tricks are needed
was able to reach 40k threads


ulim.sh
```
#!/bin/bash
# run the commands wrapped in a ulimit
cmd="$@"
user=$(whoami)
sudo bash -c "ulimit -n 102400; ulimit -i 120000; su $user -mc '$cmd'"
```


```
cat /proc/$(pgrep -x java)/limits 
echo 100000 | sudo tee /sys/fs/cgroup/pids/user.slice/user-1000.slice/pids.max
echo 120000 | sudo tee /proc/sys/kernel/threads-max
echo 600000 | sudo tee /proc/sys/vm/max_map_count
echo 200000 | sudo tee /proc/sys/kernel/pid_max

echo "UserTasksMax=100000" | sudo tee -a /etc/systemd/logind.conf
sudo systemctl status user-$UID.slice
```

cycle through localhost aliases
```
r2() { echo $((rand++%254+1)); }
declare -i rand
ulim.sh "ulimit -Sn 65536; ab -q -r -l -c20000 -n200000 http://127.0.0.$(r2):8080/json2?time=100"
```



weaknesses

doesn't test the network between the client and server
  though for a dev machine, that's not representative anyway





resources
```
https://stackoverflow.com/questions/344203/maximum-number-of-threads-per-process-in-linux

# 144GB and 72 cores
https://stackoverflow.com/questions/47838348/why-cant-i-create-50k-processes-in-linux
```

### thread usage

```
while true; do
      printf "%8d %4d \t\t %s \n" \
      	     $(pgrep -x -w java | wc -l) \
	     $(pgrep -x ab | wc -l) \
	     "$(grep MemFree /proc/meminfo)"
      sleep 1
done
```

the rampdown
```
   39999    1 		 MemFree:         3852768 kB 
   39999    1 		 MemFree:         3825196 kB 
   39999    2 		 MemFree:         3523036 kB 
   39999    0 		 MemFree:         3715016 kB 
   39999    0 		 MemFree:         3714996 kB 
   39978    0 		 MemFree:         3712220 kB 
   39884    0 		 MemFree:         3715980 kB 
   39731    0 		 MemFree:         3721572 kB 
   39702    0 		 MemFree:         3725508 kB 
   39109    0 		 MemFree:         3752272 kB 
   38482    0 		 MemFree:         3773912 kB 
   37827    0 		 MemFree:         3799844 kB 
   37141    0 		 MemFree:         3802716 kB 
   36441    0 		 MemFree:         3806972 kB 
   35688    0 		 MemFree:         3844148 kB 
   34979    0 		 MemFree:         3875072 kB 
   34253    0 		 MemFree:         3903264 kB 
   33597    0 		 MemFree:         3786520 kB 
   32884    0 		 MemFree:         3769656 kB 
   32137    0 		 MemFree:         3771432 kB 
   31372    0 		 MemFree:         3810684 kB 
   30577    0 		 MemFree:         3852032 kB 
   29783    0 		 MemFree:         3894388 kB 
   28973    0 		 MemFree:         3937328 kB 
   28140    0 		 MemFree:         3983164 kB 
   27273    0 		 MemFree:         4029548 kB 
   26399    0 		 MemFree:         4082664 kB 
   25520    0 		 MemFree:         4154560 kB 
   24699    0 		 MemFree:         4198852 kB 
   23760    0 		 MemFree:         4248948 kB 
   22799    0 		 MemFree:         4300652 kB 
   21809    0 		 MemFree:         4354452 kB 
   20782    0 		 MemFree:         4409232 kB 
   19744    0 		 MemFree:         4466776 kB 
   18665    0 		 MemFree:         4527904 kB 
   17485    0 		 MemFree:         4593956 kB 
   16247    0 		 MemFree:         4663868 kB 
   14909    0 		 MemFree:         4739424 kB 
   13482    0 		 MemFree:         4823408 kB 
   11905    0 		 MemFree:         4916412 kB 
   10132    0 		 MemFree:         5027040 kB 
    7994    0 		 MemFree:         5175884 kB 
    4951    0 		 MemFree:         5393544 kB 
      40    0 		 MemFree:         6046176 kB 
      40    0 		 MemFree:         6046744 kB 
      40    0 		 MemFree:         6047024 kB 
      40    0 		 MemFree:         6047264 kB 
```





### json2


the json2 route has a variable delay, either `Thread.sleep` or `Fiber.sleep`



```
  @RequestMapping("/json2")
  @ResponseBody
  Map<String, String>  json2(@RequestParam(required=false,defaultValue="0") Integer time) {
    if (time >= 0)
        try { Thread.sleep(time); } catch (InterruptedException ex) {}
    return Map.of("message", "Hello, World2! " + time);
  }
```

Apache Bench command line (varying `-c` argument):
```
ulim.sh "ulimit -Sn 65536; ab -r -l -c20000 -n200000 http://127.0.0.$(r2):8080/json2?time=100"
```


-c5k
```
Requests per second:    8528.68 [#/sec] (mean)
Requests per second:    8640.45 [#/sec] (mean)
Requests per second:    10669.19 [#/sec] (mean)
Requests per second:    9323.06 [#/sec] (mean)
Requests per second:    8933.16 [#/sec] (mean)
```




-c10k (warmed up)
```
Requests per second:    7585.11 [#/sec] (mean)
Requests per second:    7331.17 [#/sec] (mean)
Requests per second:    7574.36 [#/sec] (mean)
Requests per second:    7835.96 [#/sec] (mean)
Requests per second:    7286.05 [#/sec] (mean)
```


-c20k (warmed up)
```
Requests per second:    5565.48 [#/sec] (mean)
Requests per second:    4989.56 [#/sec] (mean)
Requests per second:    5298.36 [#/sec] (mean)
Requests per second:    5368.91 [#/sec] (mean)
```


-c40k cold: 2900 rps (ab usage approx 40% core)

-c40k (sum of 2 instances, warm)
```
Requests per second:    2366.06 [#/sec] (mean)
Requests per second:    2232.39 [#/sec] (mean)
total:                  4598
```


```
    { add("/json2?time",this::json2); }
    Object json2(String timeString) throws Pausable {
        int time = 0;
        try { time = Integer.parseInt(timeString); } catch (Exception ex) {}
        kilim.Task.sleep(time);
        return Map.of("message","Hello, World!" + time);
    }
```

-c20k kilim (ab uses 100% core)
```
Requests per second:    17155.85 [#/sec] (mean)
```

-c40k kilim (sum of 2, ab uses 200% core)
```
Requests per second:    12433.47 [#/sec] (mean)
Requests per second:     9144.19 [#/sec] (mean)
total:                   21577
```



-c20k spring deferred, limited to a single tomcat thread (200% core usage)
```
Requests per second:    4353.23 [#/sec] (mean)
```

2 threads (400% core usage)
```
Requests per second:    5406.01 [#/sec] (mean)
```


### HDD non-linear performance

* 10% more data results in 1000% slower performance
* ie, 100M rows to 110M rows
* linear scan first to cache data, ie simulate steady state performance

```
> size=100000000

> time pg -c "select sum(randomnumber) from world where id<$size;"
real	0m57.284s
real	0m28.485s
real	0m19.289s
real	0m18.725s


> for ii in {1..10}; do r2; ab -q -r -l -c100 -n20000 "http://127.0.0.$(r2):8080/db2?max=$size" | grep ^Req; done
Requests per second:    9747.99 [#/sec] (mean)
Requests per second:    10369.37 [#/sec] (mean)
Requests per second:    10302.12 [#/sec] (mean)
Requests per second:    9271.53 [#/sec] (mean)
Requests per second:    8961.31 [#/sec] (mean)
Requests per second:    8657.96 [#/sec] (mean)
Requests per second:    10352.47 [#/sec] (mean)
Requests per second:    10605.18 [#/sec] (mean)
Requests per second:    10323.42 [#/sec] (mean)
Requests per second:    10871.96 [#/sec] (mean)


> size=110000000

> time pg -c "select sum(randomnumber) from world where id<$size;"
real	0m35.568s
real	0m50.860s
real	0m38.134s
real	0m45.763s

> for ii in {1..10}; do r2; ab -q -r -l -c100 -n20000 "http://127.0.0.$(r2):8080/db2?max=$size" | grep ^Req; done
Requests per second:    835.00 [#/sec] (mean)
Requests per second:    1060.62 [#/sec] (mean)
Requests per second:    1027.42 [#/sec] (mean)
Requests per second:    994.84 [#/sec] (mean)
Requests per second:    994.46 [#/sec] (mean)
Requests per second:    1056.78 [#/sec] (mean)
Requests per second:    1041.08 [#/sec] (mean)
Requests per second:    1026.06 [#/sec] (mean)
Requests per second:    950.40 [#/sec] (mean)
Requests per second:    987.54 [#/sec] (mean)


> size=1000000000
Requests per second:    24.93 [#/sec] (mean)
```






### comparing blocking and fiber APIs


spring - blocks thread till query is finished
```
  @RequestMapping("/fortunes")
  List<Fortune> fortunes() {
    var fortunes =
        jdbcTemplate.query(
            "SELECT * FROM fortune",
            (rs, rn) -> new Fortune(rs.getInt("id"), rs.getString("message")));
    return fortunes;
  }
```

spring async - forks a fiber (many fibers run in a single thread) and returns immediately
```
    @RequestMapping("/fortunes")
    Object fortunes() {
        DeferredResult result = new DeferredResult<>();
        kilim.Task.fork(() -> {
            var list = db4j().submit(txn
                    -> fortunes().getall(txn).vals()
            ).await().val;
            result.setResult(list);
        });
        return result;
    }
```


kilim with db4j - the server is a fiber
```
    { add("/fortunes",this::fortunes); }
    Object fortunes() throws Pausable {
        var fortunes = db4j().submit(txn
                -> fortunes().getall(txn).vals()
        ).await().val;
	return fortunes;
    }
```


for 1000 concurrent requests
* spring: 1000 server threads + the database
* deferred: 4 server threads + 4 fiber threads + the database
* kilim: 4 server threads + the database



### node.js example

```
app.post('/users', function (req, res, next) {
  const user = req.body

  pg.connect(conString, function (err, client, done) {
    if (err) {
      // pass the error to the express error handler
      return next(err)
    }
    client.query('INSERT INTO users (name, age) VALUES ($1, $2);', [user.name, user.age], function (err, result) {
      done()

      if (err) {
        // pass the error to the express error handler
        return next(err)
      }

      res.send(200)
    })
  })
})

```





