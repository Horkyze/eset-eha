
## The query
```sql
SELECT COUNT( * ) , CONCAT( VERSION(),FLOOR(RAND()*2) ) x
FROM information_schema.tables
GROUP BY x;
```
Can result in 2 different outputs:
```
ERROR 1062 (23000): Duplicate entry '5.1.53-community-log1' for key 'group_key'
```
OR
```
+----------+-----------------------+
| count(*) | x                     |
+----------+-----------------------+
|      157 | 5.1.53-community-log0 |
|      138 | 5.1.53-community-log1 |
+----------+-----------------------+
```
The question is to explain why. First let us review some MySQL facts.

### Review of MySQL facts
* Query is executed from the inner-most function
* `GROUP BY x` -> `x` has to be unique
* `RAND(N)` without argument return pseudo-random numbers, with argument N, uses N as a seed, so each time you call RAND(7), you get the same number. 
* `FLOOR( RAND( ) *2 )` can return `0` or `1`
* `VERSION()` returns the version string of current MySQL, eg.: `5.1.53-community-log`
* `SELECT count(*) AS count` is the same as `SELECT count(*) count`, meaning the word `AS` is just "syntax sugar" and can be omitted. 

### PoC
To understand the query lets run it without the `GROUP BY x`
```sql
SELECT COUNT( * ) , CONCAT( VERSION( ) , FLOOR( RAND( ) *2 ) ) x
FROM information_schema.tables;

+----------+-----------------------+
| count(*) | x                     |
+----------+-----------------------+
|      295 | 5.1.53-community-log1 |
+----------+-----------------------+
```
Without `GROUP BY x` we can only get one kind of result as shown above. `CONCAT( VERSION( ) , FLOOR( RAND( ) *2 ) )` is evaluated only once. No problem, everything works fine.


By adding `GROUP BY x` to the query, `CONCAT( VERSION( ) , FLOOR( RAND( ) *2 ) )` is evaluated as many times as `count(*)`, so in this case it is 295. For some reason (a bug perhaps?) some sequences of `RAND()` cause group by to return error, thus revealing the version string in error message. For example `RAND(0)` always returns error, while `RAND(1)` never does. `RAND()` chooses unknown seed value, so sometime it fails and sometimes it doesn't. The probability is around 50%.

## Experiment
Why does the query for some N in `RAND(N)` fail and for others doesn't? I wrote a simple script to verify Ns, for which the query returns error. With the hope of getting some relevant information and perhaps find a correlation between the sequences of generated numbers. The script return a number N and corresponding sequence of "random" values produced by sequential RAND(N) calls. The length of the sequence is 295, because there were 295 rows in the `information_schema.tables` table.

### Results

Example output from the script.
```
0 	 0110110011101110100011101100001001101100011100100000000100100001010000001011111011001001111111011001000011110000101100101001110000010000011000110100001011010001010010001001110001101000010001010001011011100011111111001000101010111010000100010001110010110100110010101011010100110010110000100010100 
4 	 0110111111011100001011001101111101001100100010011101100111011001101101001111100010010100101011011111001001011001110011111110000110010001000000101110101000001110010110100000111110100101110100101101011000100000110100011100010001100001110000111110000101001010010011100100110111011011101010011010101 
... few line omitted ...

48 	 0111010011001010001100001010111010110001001000001000100000000111011111101000101001100110001000110011110110100100010011011111101010001011010100111100111000111111100010000000011110010110000000111011110001011111010010001100101111101110011111010001110110001000110000101011100101001000100111101110011 
49 	 0101111000001110110101110111101111010101101100110100001100001001101001101001110000110110001011001010011011011011111011011001010100011011010110001010011101011101000111100011001001110001110100010001100101111111100011110000100100010110101101111101110011001010001110100101000011110011101001100100111 
Failure probability = 32/50 = 0.640000 
```

The results are inconclusive, its hard to tell what these numbers have in common.
Remember that these numbers trigger the MySQL error, but we dont know why. Maybe a some kind of a bug.
* https://bugs.mysql.com/bug.php?id=58081
* https://bugs.mysql.com/bug.php?id=60808


### Used Script
```php
<?php 
$link = mysql_connect('localhost','root',''); 
if (!$link) { 
	die('Could not connect to MySQL: ' . mysql_error()); 
} 
echo "<pre>";

function verify_failure($i){
	$q = "SELECT COUNT( * ) , CONCAT( VERSION(),FLOOR(RAND($i)*2) ) x FROM information_schema.tables GROUP BY x;";
	$r = mysql_query($q);
	if (!$r)
		return true;
	return false;
}

$MAX_I = 1000;

$total_failures=0;
for ($i = 0; $i < $MAX_I; $i++){
	// get the generated integers using RAND($i)
	$q = "SELECT GROUP_CONCAT(rnd.x SEPARATOR '') FROM (SELECT  FLOOR(RAND($i)*2)x  FROM information_schema.tables) as rnd;";
	$r = mysql_query($q);
	$data = mysql_fetch_array($r);

	// if injection results in sql error, print the list of generated 'random' numbers by RAND($i) 
	if (verify_failure($i)){
		print("$i \t $data[0] <br>");
		$total_failures++;
    }
}

printf("<h2>Failure probability = $total_failures/$MAX_I = %f", $total_failures/$MAX_I);

?> 
```

### Sources
https://stackoverflow.com/questions/11787558/sql-injection-attack-what-does-this-do
https://security.stackexchange.com/questions/89341/sql-injection-explain-this-query

