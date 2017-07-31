## Research setting SQL Log (Turn on/off SELECT, INSERT, UPDATE, DELETE, ...)

* No setting or gem to turn on/off sql log
* Customize logger format and filter the log message with keywords (SELECT, INSERT, UPDATE, DELETE) to not display on console

### Setting
* Create `custom_formatter.rb` in folder lib

	```
	class CustomFormatter < Logger::Formatter
	  SEVERITY_TO_TAG_MAP     = {'DEBUG'=>'DEBUG', 'INFO'=>'INFO', 'WARN'=>'WARN', 'ERROR'=>'ERROR', 'FATAL'=>'FATAL', 'UNKNOWN'=>'UNKNOWN'}
	  SEVERITY_TO_COLOR_MAP   = {'DEBUG'=>'0;37', 'INFO'=>'32', 'WARN'=>'33', 'ERROR'=>'31', 'FATAL'=>'31', 'UNKNOWN'=>'37'}
	  USE_HUMOROUS_SEVERITIES = true
	
	  def initialize(arr)
	    @arr = arr
	  end
	
	  def call(severity, time, progname, msg)
	    if USE_HUMOROUS_SEVERITIES
	      formatted_severity = sprintf("%-3s","#{SEVERITY_TO_TAG_MAP[severity]}")
	    else
	      formatted_severity = sprintf("%-5s","#{severity}")
	    end
	
	    formatted_time = time.strftime("%Y-%m-%d %H:%M:%S.") << time.usec.to_s[0..2].rjust(3)
	    color = SEVERITY_TO_COLOR_MAP[severity]
	
	    @arr.any? do |item|
	      puts "\033[0;37m#{formatted_time}\033[0m [\033[#{color}m#{formatted_severity}\033[0m] #{msg.strip} (pid:#{$$})\n" unless msg.include? item
	    end
	  end
	end
	```

* Call it in `config/environments/development.rb`

	```config.log_formatter = CustomFormatter.new(%w[SELECT INSERT])```

> NOTE: `%w[SELECT INSERT]` is an array contain keywords that use filter to not display on console