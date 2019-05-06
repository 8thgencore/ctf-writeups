# Programming - Crunchy

> Trade 500 billion years of CPU time and 50 exabytes of RAM for a shiny flag : [crunchy](https://github.com/axelmaker/ctf-writeups/blob/master/INShAck2019/Crunchy/crunchy.py)

When we look at the function, we see that the value is returned to us  
```return 6 * crunchy(n - 1) + crunchy(n - 2)```  
The first thought is Fibonacci. One of the coefficients is multiplied by 6.  
To simplify the solution used [Pisano period](https://en.wikipedia.org/wiki/Pisano_period)  
Implement this function  

```python
n = 17665922529512695488143524113273224470194093921285273353477875204196603230641896039854934719468650093602325707751568
m = 100000007

def getSequencePeriod(m):
	s = []
	s.append(0)
	s.append(1)
	for i in range(2, m*6):
		s.append((6 * s[i-1] + s[i-2]) % m)
		if (s[i] == 1 and s[i-1] == 0):
			break
	return s
	
def getFibonacciRest(n, m):
	s = getSequencePeriod(m)
	period = len(s) - 2
	val = n % period
	return(s[val])

print(getFibonacciRest(n,m) # 41322239
```
Flag `INSA{41322239}`.
