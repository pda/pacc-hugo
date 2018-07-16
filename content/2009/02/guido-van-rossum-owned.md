---
title: A quote from Guido van Rossum
date: 2009-02-23 22:11:04 +1100
---

> Although I had hoped to provide something similar in Python, it quickly became clear that such an approach would be impossible because there was no way to elegantly distinguish instance variables from local variables in a language without variable declarations.

&#8212; Guido van Rossum, <a href="http://python-history.blogspot.com/2009/02/adding-support-for-user-defined-classes.html">The History of Python: Adding Support for User-defined Classes</a>

```ruby
# The Greeter class
class Greeter
  def initialize(name)
    @name = name.capitalize
  end 
  def salute
    puts "Hello #{@name}!"
  end
end
```
&#8212; Ruby example, <a href="http://www.ruby-lang.org/">ruby-lang.org front page</a>

Owned.
