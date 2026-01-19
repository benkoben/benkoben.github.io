
# Rasapberry Pi Pico and NTP

Whenever I have time I like to work on a little side project of mine. A small alarmclock that I build from scratch to learn about electronics and microcontrollers. I've come quite far so far. My last commitment has been to retrieve time from an NTP server and automatically adjust the time with the correct timezones. 

One of the things I really have enjoyed with this project is the mix of high level programming and low level programming. Since I am using micropython to program the Raspberry Pi pico I have lots of built-in abstractions ready at my hands. But for this particular task I had to dive a bit into the less shallow part of the pool and explore how time is retrieved from NTP.

Since I will be eventually be using my alarmclock as a daily driver and considering the fact that I live in Sweden, I want it to be able to adjust itself for daylight saving time adjustment. In this case CET+1 (aka. winter time) and CEST+2 (aka. summertime).

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
			yearday, # TODO: I need to figure out the yearday of this particular sunday
        ))

	"""
		_last_sunday_in_march returns the current year's last sunday in March as a Datetime object
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
		_last_sunday_in_march returns the current year's last sunday in October as a Datetime object
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

			# TODO: what does the transmit timestmap mean?
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
