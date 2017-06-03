# LeetCode算法题《Word Break》——字典匹配，动态规划

**题目**：

Given a string s and a dictionary of words dict, determine if s can be segmented into a space-separated sequence of one or more dictionary words.

For example, given 
s = `"leetcode"` , 
dict = `["leet", "code"]` .

Return true because `"leetcode"` can be segmented as `"leet code"`.

**解析**：

题目的大意是判断一个字符串能否切割成若干个子串，每个子串都是字典中的元素。

这道题是典型的动态规划算法题。主要思路如下：

1. 概念“能被拆分”代表：字符串s能够拆分成若干子串，使得每一个子串都是字典dict中的元素。
2. 将字符串s[0,i)从j处分成两部分s[0,j)和s[j,i)。
3. 如果s[j,i)是字典中的元素，并且s[0,j)能被拆分，显然s[j,i)能被拆分。
4. 显然，假设字符串s的长度为N，如果s[0,N)能够被拆分，说明字符串s能够被拆分。

该题目的转移条件：

F[0, N) = F[0, i) + F[i, j) + F[j, N)

**解决方法**：

定义一个布尔量数组`OK[]`，数组长度为N+1 ，如果`OK[i] == TRUE;`表示子串s[0, i)能够被拆分。OK[0]无意义，设为为TRUE。

显然，如果`OK[N] == TRUE`，则字符串s能够被拆分。

字符串数组：

| 0    | 1    | 2    | 3    | 4    | 5    | 6    | 7    |
| ---- | ---- | ---- | ---- | ---- | ---- | ---- | ---- |
| l    | e    | e    | t    | c    | o    | d    | e    |

OK数组：

| 0    | 1    | 2    | 3    | 4    | 5    | 6    | 7    | 8    |
| ---- | ---- | ---- | ---- | ---- | ---- | ---- | ---- | ---- |
| true |      |      |      |      |      |      |      |      |

匹配过程：

| 过程   | 拆分                                       | 结果         |
| ---- | ---------------------------------------- | ---------- |
| 1    | "l"                                      | 均不匹配       |
| 2    | "1"&&"e", "le"                           | 均不匹配       |
| 3    | "1e"&&"e", "l"&&"ee", "lee"              | 均不匹配       |
| 4    | "lee"&&"t", "le"&&"et", "l"&&"eet", <u>"leet"</u>, break | OK[4]=TRUE |
| 5    | "leet"&&"c", "lee"&&"ct", "le"&&"ect", "l"&&"eetc", "leetc" | 均不匹配       |
| 6    | "leetc"&&"o", "leet"&&"co", "lee"&&"tco", "le"&&"etco", "l"&&"eetco", "leetco" | 均不匹配       |
| 7    | "leetco"&&"d", "leetc"&&"od", "leet"&&"cod", "lee"&&"tcod", "le"&&"etcod", "l"&&"eetcod" | 均不匹配       |
| 8    | "leetcod"&&"e", "leetco"&&"de", "leetc"&&"ode", <u>"leet"&&"code"</u>, break | OK[8]=TRUE |

**代码**

```cpp
bool wordBreak(string s, unordered_set<string> &dict)
{
	int N = s.length();
  	vector<bool> OK(N+1, false);
  	OK[0] = true;
  	int i, j;
  	for(i = 1; i < N+1; i++)
    {
      	for(j = i-1; j >=0; j--)
        {
          	if(OK[j] && dict.find(s.substr(j, i-j)) != dict.end())
            {
              	OK[i] = true;
              	break;
            }
        }
    }
  	return OK[N];
}
```



