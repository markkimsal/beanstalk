# beanstalkd

[![Build Status](https://img.shields.io/travis/amphp/beanstalkd/master.svg?style=flat-square)](https://travis-ci.org/amphp/beanstalkd)
[![CoverageStatus](https://img.shields.io/coveralls/amphp/beanstalkd/master.svg?style=flat-square)](https://coveralls.io/github/amphp/beanstalkd?branch=master)
![Unstable](https://img.shields.io/badge/api-unstable-orange.svg?style=flat-square)
![License](https://img.shields.io/badge/license-MIT-blue.svg?style=flat-square)

`amphp/beanstalkd` is a non-blocking beanstalkd client for use with the [`amp`](https://github.com/amphp/amp)
concurrency framework.

**Required PHP Version**

- PHP 7.0+

**Installation**

```bash
composer require amphp/beanstalk
```

**Example**

```php
include ('vendor/autoload.php');
Amp\run(function () {
	//you must put the port number, there's no default port generated.
	//this connection string is passed directly to parse_uri, which treats
	//"127.0.0.1" as a path and not a host.
	$client = new Amp\Beanstalk\BeanstalkClient('127.0.0.1:11300');

	$client->watch('input');
	Amp\repeat(function() use ($client) {

		$outbuffer = '';
		$promise = $client->reserve(0);  //return immediately if there are no jobs

		$promise->when( function($error, $result, $cbData) use ($client, &$outbuffer) {

			if ($error instanceOf Amp\Beanstalk\TimedOutException) {
				//this is normal for reserve-with-timeout requests
				return;
			}
			if ($error instanceOf Amp\Beanstalk\DeadlineSoonException) {
				//if you aren't deleting jobs on time, you'll get deadlinesoon exceptions
				return;
			}
			if ($error instanceOf Amp\Beanstalk\ConnectException) {
				//you may get connection exceptions after you try to reserve
				return;
			}

			if ($result) {
				echo "RESERVED JOB: ".$result[0]."\n";
			}
			if (!$result) {
				//some kind of unhandled exception
				print_r( get_class($error));
				return;
			}

			$outbuffer .= $result[1];
			try {
				$id = $result[0];
				$k  = $client->delete($id);
				$k->when( function($err, $res) use ($client, $id) {
					if ($err) {
						//!result[0] ID was not deleted
					}
				});
			} catch (Exception $e) {
			}
		});
	}, $msInterval=50);
});
```
