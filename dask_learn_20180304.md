# Dask 和 pandas 的区别  
1. Dask的许多操作都是延迟操作，只有在触发compute()的时候才会真正执行。  
2. Dask的并没有完全复制pandas的的所有操作，例如不能读写excel的的文件，不能多谢压缩文件等等。  

# Dask的读写操作  

``` 
import dask.dataframe as dd  
df = dd.read_csv('taxi/*.csv', assume_missing=True)  
df['datetime'] = dd.to_datetime(df['datetime'])  
df['hour'] = df['datetime'].dt.hour  
```
以上的这些操作都和在pandas下一致，除了dask支持taxi/*.csv，类似这种读取模式，dask支持自动将这些csv连接起来。当然，如果想要手动也可以，pd.concat()以及dd.concat()都可以完成这个操作。  

# Dask 的 dataframe 注意事项  
1. 惰性求值，所有的操作都是惰性求值，sum()，mean()，等等操作如果要得到结果，就必须在后面跟上compute()，这样才能得到结果。  
2. @delayed装饰器的使用，Dask会自动帮你完成任务调度(刚开始似乎还不需要深入理解这个地方)。将想要延迟(或者说惰性求值)的函数加上这个装饰器，Dask就会自己去查看函数中的依赖关系，并在有compute()的情况下触发这个函数。(目前的我的理解是这样的)  


# 处理文字类的数据  
dask提供了dask这个工具，可以用来处理文字类的统计工作。 

``` 
import dask.bag as db  
df = db.read_text()  
df.count().compute() #统计一共有多少字
df.take(1)  
df.take(1)[60]  
df.str.split(' ')  
df.str.split(' ').map(len)  
df.str.split(' ').map(len).mean().compute()  
```  

除了这些，还有高级的用法，比如filter  

``` 
lower = speeches.str.lower()  
health = lower.filter(lambda s: 'health care' in s)  
n_health = health.count().compute()  
```  

``` 
bill = db.read_text(congress/bills*.json)  
bill_dict = bill.map(json.loads)  
bill_first = bill_dict.take(1)  
```  
# 一些通常情况的处理  
1. 把无法放进内存的操作，加一个@delayed装饰器  
2. 在for循环下使用延迟计算的函数处理数据  
3. 使用compute()触发计算  

``` 
@delayed
def read_flights(filename):

    # Read in the DataFrame: df
    df = pd.read_csv(filename, parse_dates=['FL_DATE'])

    # Calculate df['WEATHER_DELAY']
    df['WEATHER_DELAY'] = df['WEATHER_DELAY'].replace(0, np.nan)

    # Return df
    return df
    
# Loop over filenames with index filename
for filename in filenames:
    # Apply read_flights to filename; append to dataframes
    dataframes.append(read_flights(filename))

# Compute flight delays: flight_delays
flight_delays = dd.from_delayed(dataframes)

# Print average of 'WEATHER_DELAY' column of flight_delays
print(flight_delays['WEATHER_DELAY'].mean().compute())
```
以上就是通用的一个过程，下面是另一个例子  

```
is_snowy = weather['Events'].str.contains('Snow').fillna(False)

# Create filtered DataFrame with weather.loc & is_snowy: got_snow
got_snow = weather.loc[is_snowy]

# Groupby 'Airport' column; select 'PrecipitationIn'; aggregate sum(): result
result = got_snow.groupby('Airport')['PrecipitationIn'].sum()

# Compute & print the value of result
print(result.compute())
```

