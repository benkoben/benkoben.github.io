
# Retrieving NTP time on a Raspberry Pi Pico

Whenever I have time I like to work on a little side project of mine. A small alarmclock that I build from scratch to learn about electronics and microcontrollers. One of the problems I had to solve was time retrieval, specifically retrieving time from an NTP server. The requirements are as follows:

1. Allow the alarmclock to sync its RTC clock with an NTP time source
2. Automatically adjust for day light time savings (CET+1 and CEST+2)

One of the things I really have enjoyed with this project is the mix of high level programming and low level programming. Although I am using micropython to program the Raspberry Pi pico which has lots of nice features and built-in abstractions. But for this particular task I had to dive a bit into the nitty gritty details of NTP.

# The code

```python
import usocket as socket
import struct
import time

from datetime import date

from config import NetworkConfig

_tz = {
	"CEST": +2, # Summetime
	"CET": +1
}

"""
Ntp fetches the current time from host upon initialization.
"""
class Ntp():
	def __init__(self, host):
		
		# Fetch UTC time from NTP server
		self.current_time = time.gmtime(
			self.fetch_current_time(host)
        )
		
	def year(self):
        #     year includes the century (for example 2014).
		return self.current_time[0]
        
	def month(self):
        #     month is 1-12
		return self.current_time[1]
	
	def monthday(self):
        #     monthday is 1-31
		return self.current_time[2]
	
	def hour(self):
        #     hour is 0-23
		return self.current_time[2]

	def minute(self):
        #     minute is 0-59
		return self.current_time[4]
	
	def second(self):
        #     second is 0-59
		return self.current_time[5]
	
	def weekday(self):
        #     weekday is 0-6 for Mon-Sun
		return self.current_time[6]
	
	def yearday(self):
        #     yearday is 1-366
		return self.current_time[7]

	def _is_sunday(self, year, month, day) -> bool:
		t = (year, month, day, 0, 0, 0, 0, 0)
		return time.localtime(time.mktime(t))[6] == 6

	"""
		_sunday returns a sunday as a epoch timestamp
	"""	
	def _sunday(self, year, month, day) -> int:
		yearday = date(year, month, day).timetuple()[-2]
		return time.mktime((
			year,
			month,
			day,
			0, # hour
			0, # minute
			0, # seconds
			6, # weekday
			yearday,
        ))

	"""
		_last_sunday_in_march returns the current year's last sunday in March as a 8-tuple datetime object
	"""
	def _last_sunday_in_march(self) -> int:
		# Loop trough the last week in march and find sunday
		year = self.year()
		for day in range(31, 24, -1):
			if self._is_sunday(year, 3, day):
				return int(self._sunday(
					year, 3, day
				))
		return 0

	"""
		_last_sunday_in_march returns the current year's last sunday in October as a 8-tuple datetime object
	"""
	def _last_sunday_in_october(self) -> int:
		# Loop trough the last week in october and find sunday
		year = self.year()
		for day in range(31, 24, -1):
			if self._is_sunday(year, 10, day):
				return int(self._sunday(
					year, 10, day
				))
		return 0

	def adjust_for_daylight_savings(self):
		new_ts = time.mktime(self.current_time)
		current_epoch = time.mktime(self.current_time)
		if current_epoch > self._last_sunday_in_march() and current_epoch < self._last_sunday_in_october():
			# Adjust the hour
			new_ts += _tz["CEST"] * 3600
		else:	
			new_ts += _tz["CET"] * 3600
		self.current_time = time.gmtime(new_ts)
			
		
	def fetch_current_time(self, NTP_HOST: str) -> int:
		# Used to convert the received transmit_timestamp to unix time.
		NTP_DELTA = 2208988800
		# Create a 48 byte packet
		NTP_QUERY = bytearray(48)
		# First byte (LI=0, VN=3, Mode=3 for client mode)
		NTP_QUERY[0] = 0x1B
		
		UNIX_TIME=""
		# Set the target address and port
		addr = socket.getaddrinfo(NTP_HOST, 123)[0][-1]
		# Create a datagram UDP socket
		s = socket.socket(socket.AF_INET, socket.SOCK_DGRAM)
		try:
			s.settimeout(1)
			s.sendto(NTP_QUERY, addr)
			msg = s.recv(48)

			print(f"Response recieved: {msg}")
			# The response is also a 48 byte packet where the last 8 bytes contain the "transmit timestamp"
			transmit_timestamp = struct.unpack("!I", msg[40:44])[0]

			print(f"Transmit timestamp: {transmit_timestamp}")
			UNIX_TIME = transmit_timestamp - NTP_DELTA

		except socket.timeout:
			print("Timeout: No response from NTP server")
		except Exception as e:
			print(f"Error occurred: {e}")
		finally:
			s.close()

		return UNIX_TIME
	

if __name__ == "__main__":
	now = Ntp(host="pool.ntp.org")

	# In UTC format
	print("timestamp recieved from NTP")
	print(now.current_time)

	# Adjust for current timezone
	print("timestamp after timezone adjustment (winter time)")
	# Expect 1 hour shift since its currenly january
	now.adjust_for_daylight_savings()
	print(now.current_time)

	# Adjust the date to test summertime as well
	print("adjust timestamp to timestamp (summertime)")
	now.current_time = time.gmtime(1753213285)
	print("before summertime adjustment")
	print(now.current_time)

	now.adjust_for_daylight_savings()
	print("after summertime adjustment")
	print(now.current_time)
```

I have defined a class which fetches time whenever its instantiated. The current time is saved as an 8-tuple and in memory with the `self.current_time` attribute. I added lots of getter methods to the `Ntp` class which allow me to quickly retrieve different parts of this 8-tuple (year, month, day etc.).

## Daylight timesavings

To adjust for daylight time savings I implemented a bunch of internal/private methods to determine wether or not `self.current_time` falls between the last sunday in march and the last sunday in october (in other words summertime, a.k.a CEST+2). 

The first method `_is_sunday()` checks wether or not a given timestamp actually is a sunday (the 6th week day). This method is used by both `_last_sunday_in_march()` and `_last_sunday_in_october()` when they loop through all the last weekdays in both these months respectively to determine which one of them is a sunday. When a sunday is found, an epoch timestamp is constructed with the `_sunday()` method for that particular sunday.

## NTP time retrieval

As for fetching the time itself, the `fetch_current_time()` method is used. It does the following:

1. First a NTP request is constructed and a delta is defined. This is a 48-byte packet sent to the NTP server on port 123.
2. A network socket is defined with details about the remote NTP server, this is used to send/write the packet onto and read the response from.
3. Once the response is received, it is transformed from a so called "Transmit timestamp" into a UNIX timestamp.
4. The UNIX_TIMESTAMP is returned and because its in epoch format, the built-in `time` library understands it and we can use `time.gmtime()` to convert that into the 8-tuple I mentioned earlier.

To demonstrate lets run the code within the if clause at the end.

```
~/Documents/projects/wake-up-alarm$ mpremote run ntp.py
Response recieved: b'\x1c\x03\x00\xe6\x00\x00\x00J\x00\x00\x00\x10\nA\x08\x84\xed,\xcb\xb6]\x12S\xc3\x00\x00\x00\x00\x00\x00\x00\x00\xed,\xcc\x04V-KY\xed,\xcc\x04V2\xc7\xa6'
Transmit timestamp: 3979136004
UNIX timestamp recieved from NTP:  1770147204
8-tuple format:  (2026, 2, 3, 19, 33, 24, 1, 34)
timestamp after timezone adjustment (winter time)
(2026, 2, 3, 20, 33, 24, 1, 34)
adjust timestamp to timestamp (summertime)
before summertime adjustment
(2025, 7, 22, 19, 41, 25, 1, 203)
after summertime adjustment
(2025, 7, 22, 21, 41, 25, 1, 203)
```

> I used a hardcoded value to test the summertime adjustments, hence the different years.

# Conclusion

I now have a small class that allows me to retrieve the time from a remote NTP server. The class itself makes formatting time really easy since it allows me to quickly access different parts of the timestamp (year, month, day etc.). While my alarmclock only renders hours and minutes, it still needs to be aware of the time as a whole, which is possible by syncing the timestamp to the Raspberry Pi Pico internal RTC battery clock. Syncing the received timestamp can easily be accomplished via the builtin [`RTC.datetime()`](https://docs.micropython.org/en/latest/library/machine.RTC.html) method which expects a 8-tuple format.


And now I will hopefully never be late again. Thanks for reading my first ever post!
