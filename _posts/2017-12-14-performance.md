测试分成cli模式和web模式。测试环境：
- centos6.4
- x86_64
- 4  Intel(R) 2.10GHz
- php版本分别为5.6.30和7.1.12


# CLI模式

## 高耗内存脚本
50万数据赋值
```php
<?php
ini_set('memory_limit', '-1');
$stratTime   = microtime(true);
$startMemory = memory_get_usage();
$a = array();
for($i = 1; $i <= 500000; $i++){
        $a[$i] = $i;
}
echo $a[1]."\n";
$endTime    = microtime(true);
$runtime    = ($endTime - $stratTime) * 1000; //将时间转换为毫秒
$endMemory  = memory_get_usage();
$usedMemory = ($endMemory - $startMemory);
echo "运行时间: {$runtime} 毫秒\n";
echo "耗费内存: {$usedMemory} byte\n";
```

测试结果：
| 测试项        | php5      |  php7    |
| -------      | -----:     | :----:  |
| 时间(ms)      | 99.92     | 22.39    |
| 内存(byte)    |72195240   |16781424  |

## 高cpu脚本
20000随机数据快速排序
```php
<?php
function QuickSort($arr,$low,$high)
{
 if($low>$high)
   return ;
 $begin=$low;
 $end=$high ;
 $key=$arr[$begin];
 while($begin<$end)
 {
    while($begin<$end&&$arr[$end]>=$key)
       --$end ;
    $arr[$begin]=$arr[$end];
    while($begin<$end&&$arr[$begin]<=$key)
      ++$begin;
    $arr[$end]=$arr[$begin];

 }
  $arr[$begin]=$key;
  QuickSort($arr,$low,$begin-1);
  QuickSort($arr,$begin+1,$high);
}
$stratTime   = microtime(true);
$startMemory = memory_get_usage();
$arr=array();
for($i=0;$i<20000;$i++)
{
 array_push($arr,rand(1,20000));
}
QuickSort($arr,0,19999);
$endTime    = microtime(true);
$runtime    = ($endTime - $stratTime) * 1000; //将时间转换为毫秒
$endMemory  = memory_get_usage();
$usedMemory = ($endMemory - $startMemory);
echo "运行时间: {$runtime} 毫秒\n";
echo "耗费内存: {$usedMemory} byte\n";
?>
```

测试结果：
| 测试项        | php5      |  php7    |
| -------      | -----:     | :----:  |
| 时间(ms)      | 31690     | 7058    |
| 内存(byte)    | 2983080   | 1052752  |

在使用cli模式时，php7无论在执行时间和内存消耗的表现上都比php5有大幅提升。

# web 模式
使用yii2框架测试，分别测试渲染页面和api接口下两者性能差别。

## php5下
首先测试yii2有渲染页面下的结果，使用apache bench模拟并发量100的1000次请求：
```shell
ab -c100 -n1000 http://xxx.com/aaaa/index
```
测试结果：
```shell
Server Software:        nginx
Server Hostname:        xxx.com
Server Port:            80

Document Path:          /aaaa/index
Document Length:        407979 bytes

Concurrency Level:      100
Time taken for tests:   110.025 seconds
Complete requests:      1000
Failed requests:        0
Write errors:           0
Total transferred:      408568000 bytes
HTML transferred:       407979000 bytes
Requests per second:    9.09 [#/sec] (mean)
Time per request:       11002.541 [ms] (mean)
Time per request:       110.025 [ms] (mean, across all concurrent requests)
Transfer rate:          3626.36 [Kbytes/sec] received

Connection Times (ms)
              min  mean[+/-sd] median   max
Connect:        0    0   0.4      0       2
Processing:  1396 10477 1547.9  10745   14127
Waiting:     1339 10263 1524.7  10522   13865
Total:       1396 10478 1547.6  10745   14127

Percentage of the requests served within a certain time (ms)
  50%  10745
  66%  10974
  75%  11100
  80%  11174
  90%  11383
  95%  11670
  98%  12445
  99%  12856
 100%  14127 (longest request)
```

接下来测试api接口：
```shell
ab -c100 -n1000 http://xxx.com/bbbb/cccc
```
测试结果：
```shell
Server Software:        nginx
Server Hostname:        xxx.com
Server Port:            80

Document Path:          /bbbb/cccc
Document Length:        1091 bytes

Concurrency Level:      100
Time taken for tests:   10.642 seconds
Complete requests:      1000
Failed requests:        0
Write errors:           0
Total transferred:      1240000 bytes
HTML transferred:       1091000 bytes
Requests per second:    93.97 [#/sec] (mean)
Time per request:       1064.184 [ms] (mean)
Time per request:       10.642 [ms] (mean, across all concurrent requests)
Transfer rate:          113.79 [Kbytes/sec] received

Connection Times (ms)
              min  mean[+/-sd] median   max
Connect:        0    0   0.4      0       3
Processing:    50 1022 191.7   1067    3359
Waiting:       48 1022 191.8   1067    3359
Total:         51 1022 191.4   1067    3361

Percentage of the requests served within a certain time (ms)
  50%   1067
  66%   1082
  75%   1092
  80%   1096
  90%   1112
  95%   1128
  98%   1146
  99%   1163
 100%   3361 (longest request)
```

## php7下

测试render页面
```shell
Server Software:        nginx
Server Hostname:        xxx.com
Server Port:            80

Document Path:          /aaaa/index
Document Length:        407981 bytes

Concurrency Level:      100
Time taken for tests:   79.700 seconds
Complete requests:      1000
Failed requests:        0
Write errors:           0
Total transferred:      408552000 bytes
HTML transferred:       407981000 bytes
Requests per second:    12.55 [#/sec] (mean)
Time per request:       7970.044 [ms] (mean)
Time per request:       79.700 [ms] (mean, across all concurrent requests)
Transfer rate:          5005.95 [Kbytes/sec] received

Connection Times (ms)
              min  mean[+/-sd] median   max
Connect:        0    0   0.5      0       5
Processing:  1200 7562 1237.8   7710    9975
Waiting:     1164 7503 1231.2   7651    9860
Total:       1200 7562 1237.5   7710    9976

Percentage of the requests served within a certain time (ms)
  50%   7710
  66%   7954
  75%   8119
  80%   8254
  90%   8684
  95%   9051
  98%   9449
  99%   9629
 100%   9976 (longest request)
```
测试api接口:
```shell
Server Software:        nginx
Server Hostname:        xxx.com
Server Port:            80

Document Path:          /bbbb/cccc
Document Length:        1091 bytes

Concurrency Level:      100
Time taken for tests:   7.601 seconds
Complete requests:      1000
Failed requests:        0
Write errors:           0
Total transferred:      1243000 bytes
HTML transferred:       1091000 bytes
Requests per second:    131.56 [#/sec] (mean)
Time per request:       760.083 [ms] (mean)
Time per request:       7.601 [ms] (mean, across all concurrent requests)
Transfer rate:          159.70 [Kbytes/sec] received

Connection Times (ms)
              min  mean[+/-sd] median   max
Connect:        0    0   0.5      0       3
Processing:    44  730 124.2    761     837
Waiting:       42  730 124.2    761     837
Total:         44  730 123.8    761     837

Percentage of the requests served within a certain time (ms)
  50%    761
  66%    772
  75%    778
  80%    783
  90%    793
  95%    804
  98%    820
  99%    828
 100%    837 (longest request)
```

性能有所提升。