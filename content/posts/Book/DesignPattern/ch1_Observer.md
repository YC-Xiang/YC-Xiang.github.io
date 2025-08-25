# Book Notes

**The Observer Pattern**: defines a one-to-many dependency between objects so that when one object changes state, all of its dependents are notified and updated automatically.

Design Pattern:

![](https://xyc-1316422823.cos.ap-shanghai.myqcloud.com/20250818224348818.png)

# Example

![](https://xyc-1316422823.cos.ap-shanghai.myqcloud.com/20250818221522191.png)

## Publisher

需要提供注册、注销、通知观察者的接口。

WeatherData 是 publisher 的实现类。

```java
// publisher interface
public interface Subject {
	public void registerObserver(Observer o);
	public void removeObserver(Observer o);
	public void notifyObservers();
}

// publisher implementation
public class WeatherData implements Subject {
	private List<Observer> observers; // 观察者列表
	private float temperature;
	private float humidity;
	private float pressure;
	
	public WeatherData() {
		observers = new ArrayList<Observer>();
	}
	
	public void registerObserver(Observer o) { // 注册观察者
		observers.add(o);
	}
	
	public void removeObserver(Observer o) { // 注销观察者
		observers.remove(o);
	}
	
	public void notifyObservers() { // 通知观察者
		for (Observer observer : observers) {
			observer.update(temperature, humidity, pressure);
		}
	}
	
	public void measurementsChanged() { // 数据变化时，通知观察者
		notifyObservers();
	}
	
	public void setMeasurements(float temperature, float humidity, float pressure) { // 设置数据
		this.temperature = temperature;
		this.humidity = humidity;
		this.pressure = pressure;
		measurementsChanged();
	}

	public float getTemperature() { // 获取温度
		return temperature;
	}
	
	public float getHumidity() { // 获取湿度
		return humidity;
	}
	
	public float getPressure() { // 获取气压
		return pressure;
	}

}
```

## Observer

需要提供一个 update 函数，当 publisher 通知观察者时，会统一调用这个函数。

```java
// observer interface
public interface Observer {
	public void update(float temp, float humidity, float pressure);
}

public interface DisplayElement {
	public void display();
}

// observer 1
public class CurrentConditionsDisplay implements Observer, DisplayElement {
	private float temperature;
	private float humidity;
	private WeatherData weatherData;
	
	public CurrentConditionsDisplay(WeatherData weatherData) {
		this.weatherData = weatherData;
		weatherData.registerObserver(this);
	}
	
	public void update(float temperature, float humidity, float pressure) { // 更新数据
		this.temperature = temperature;
		this.humidity = humidity;
		display();
	}
	
	public void display() { // 每个 observer 都有自己不同的 display 函数，用于显示数据
		System.out.println("Current conditions: " + temperature 
			+ "F degrees and " + humidity + "% humidity");
	}
}

// observer 2
public class ForecastDisplay implements Observer, DisplayElement {
	private float currentPressure = 29.92f;  
	private float lastPressure;
	private WeatherData weatherData;

	public ForecastDisplay(WeatherData weatherData) {
		this.weatherData = weatherData;
		weatherData.registerObserver(this);
	}

	public void update(float temp, float humidity, float pressure) {
        	lastPressure = currentPressure;
		currentPressure = pressure;
		display();
	}

	public void display() {
		System.out.print("Forecast: ");
		if (currentPressure > lastPressure) {
			System.out.println("Improving weather on the way!");
		} else if (currentPressure == lastPressure) {
			System.out.println("More of the same");
		} else if (currentPressure < lastPressure) {
			System.out.println("Watch out for cooler, rainy weather");
		}
	}
}
```

test code:

```java
// test code
public class WeatherStation {
	public static void main(String[] args) {
		WeatherData weatherData = new WeatherData();
	
		CurrentConditionsDisplay currentDisplay = 
			new CurrentConditionsDisplay(weatherData);
		ForecastDisplay forecastDisplay = new ForecastDisplay(weatherData);

		weatherData.setMeasurements(80, 65, 30.4f);
		weatherData.setMeasurements(82, 70, 29.2f);
		weatherData.setMeasurements(78, 90, 29.2f);
		
		weatherData.removeObserver(forecastDisplay);
		weatherData.setMeasurements(62, 90, 28.1f);
	}
}
```

publisher 通知 observer 更新数据，上面的做法是 publisher 主动将数据传递给 observer。

还有一种做法，publisher 通知 observers 的 update 函数中，不传入任何参数。Observer 得到通知后，主动调用 publisher 提供的接口 (getTemperature/getHumidity/getPressure) 从 publisher 获取想要的数据。
