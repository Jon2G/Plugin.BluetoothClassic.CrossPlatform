![picture](https://github.com/rostislav-nikitin/BluetoothClassic.Xamarin/blob/master/documentation/images/logo_128x128.png?raw=true)
# Plugin.BluetoothClassic for Xamarin
This is a plug-in that supports transmitting/receiving data with use of the SPP (Serial Port Profile) through a bluetooth classic protocol in the next types of the Xamarin projects:
* Xamarin.Forms
* Xamarin.Android

<!-- * Xamarin.iOS (not implemented yet)
* Xamarin.UWP (not implemented yet) -->

## How to install it?
One of they ways is to use a NuGet package manager:
* `PM> Install-Package Plugin.BluetoothClassic -Version 1.0.5`

If you building Xamarin.Forms application then you need to install this package into the both: Xamarin.Forms/Xamarin.Android projects.

More information about this NuGet package is accessbile by this link: https://www.nuget.org/packages/Plugin.BluetoothClassic/

## How to use it?
Use `DependencyService.Resolve<IBluetoothAdapter>` to get instance of the device default bluetooth adapter
Use `IBluetoothAdapter` in a next way:
* Properties:
  * `bool Enabled` to check is bluetooth adapter enabled
  * `IEnumerable<BluetoothDeviceModel> BondedDevices` to show list of the bonded remote devices 
* Methods:
  * `void Enable()` to enable a default bluetooth adapter
  * `void Disable()` to disable a default bluetooth adapter
  * `IBluetoothConnection CreateConnection(BluetoothDeviceModel bluetoothDeviceModel)` to create a new connection to the bonded remote device
  * `IBluetoothManagedConnection CreateManagedConnection(BluetoothDeviceModel bluetoothDeviceModel)` to create a new managed connection to the bonded remote device

As you see there are supported two different connection types: the `IBluetoothConnection` and the `IBluetoothManagedConnection` one.

Let's start from the `IBluetoothConnection` one.

### `IBluetoothConnection: IDisposable`
Ypu can use it for the short-time transmit/recive transactions. Sometimes you need to get some small piece of data and close connection. For example you need get 5 bytes of the data each 30 minutes. You need to get it from temperature sensor connected to the MCU wich also connected to the bluetooth module.
In this case the IBluetoothConnection is your choice.
It's life-time usually a period of transmitting/reciving data and usually it wrapped into the `using(...){..}` statement to automaticlly dispose it after it's job done.

#### Members

Use `IBluetoothConnection` in a next way:
 * `Task ConnectAsync()` to connect to the remote bluetooth device asynchronously
 * `Task TransmitAsync(Memory<byte> buffer, CancellationToken cancellationToken = default)`  to transmit data to the remote bluetooth device asynchronously
 * `Task TransmitAsync(byte[] buffer, int offset, int count)` to transmit data to the remote bluetooth device asynchronously
 * `Task TransmitAsync(byte[] buffer, int offset, int count, CancellationToken cancellationToken)` to transmit data to the remote bluetooth device asynchronously
 * `bool DataAvailable { get; }` to check is any data available to recive
 * `Task<int> ReciveAsync(Memory<byte> buffer, CancellationToken cancellationToken = default)` to recive data from the remote bluetooth device asynchronously
 * `Task<int> ReciveAsync(byte[] buffer, int offset, int count)`
 * `Task<int> ReciveAsync(byte[] buffer, int offset, int count, CancellationToken cancellationToken)`

#### Example
(From the examples/Retry example)

```CSharp
using(var connection = App.BluetoothAdapter.CreateConnectionAsync(remoteDevice))
{
    if (await connection.RetryConnectAsync(retriesCount: 2))
    {

        byte[] buffer = new byte[TransmitBufferSizeDefault] { (byte)stepperDigit.Value };
        try
        {
            await connection.RetryTransmitAsync(buffer, BufferOffsetDefault, buffer.Length);
        }
        catch (Exception exception)
        {
            await DisplayAlert("Exception", exception.Message, "Close");
        }
    }
    else
    {
        await DisplayAlert("Exception", "Can not connect.", "Close");
    }
}
```
### `IBluetoothManagedConnection: IDisposable`
You can use it for the long-time bluetooth connections. For example to make you device stay connected to some other remote bluetooth device for some minutes/hours for continuous data exchange. This type of connection contains internal connection manager, that care about reconnecting if connection was lost. It use transmit queue. When some code calls Transmit method it simply put buffer data into this queue. By fact data will be transmitted by the transmitter thread when connection will be available. Also depends on settings, this type of connection listen input stream for the data.
It's life-time usually equal to the life-time of the application. If created on application starts and disposed at application shutdowns.

#### Members
* `ConnectionState IBluetoothConnection.ConnectionState` to check a connection state
* `void Connect()` to connect to the remote device
* `void IDisposable.Dispose()` to disconnect from the remote device and free unmanaged resources
* `void Transmit(Memory<byte> buffer)` to add a buffer to the transmit queue (buffers are transmitting to the remote device sequentially while remote device connected otherwise data will be stored until it will be reconnected)
* `event StateChanged OnStateChanged` to subscribe/unsubscribe on any connection state changes
* `event Transmitted OnTransmitted` to subscribe/unsubscribe on a data transmitted to the connected remote device
* `event Recived OnRecived` to subscribe/unsubscribe for/from reciving a data from the connected remote device
* `event Error OnError` to subscribe/unsubscribe on any connection errors

#### Example
(From the examples/Digit example)

```CSharp
[XamlCompilation(XamlCompilationOptions.Compile)]
public partial class SelectBluetoothRemoteDevicePage : ContentPage
{
    private readonly IBluetoothAdapter _bluetoothAdapter;

    public SelectBluetoothRemoteDevicePage()
    {
        _bluetoothAdapter = DependencyService.Resolve<IBluetoothAdapter>();
        InitializeComponent();
    }
    
    ...
    
    private async Task<bool> TryConnect(BluetoothDeviceModel bluetoothDeviceModel)
    {
        const bool Connected = true;
        const bool NotConnected = false;

        var connection = _bluetoothAdapter.CreateManagedConnection(bluetoothDeviceModel);
        try
        {
            connection.Connect();
            App.CurrentBluetoothConnection = connection;

            return Connected; 
        }
        catch (BluetoothConnectionException exception)
        {
            await DisplayAlert("Connection error",
                $"Can not connect to the device: {bluetoothDeviceModel.Name}({bluetoothDeviceModel.Address}).\n" +
                    $"Exception: \"{exception.Message}\"\n" +
                    "Please, try another one.",
                "Close");

            return NotConnected;
        }
        catch (Exception exception)
        {
            await DisplayAlert("Generic error", exception.Message, "Close");
            return NotConnected;
        }
        
        ...

    }
}

[XamlCompilation(XamlCompilationOptions.Compile)]
public partial class DigitPage : ContentPage
{
    public DigitPage()
    {
        InitializeComponent();

        DigitViewModel model = (DigitViewModel)BindingContext;
        model.PropertyChanged += Model_PropertyChanged;

        if (App.CurrentBluetoothConnection != null)
        {
            App.CurrentBluetoothConnection.OnStateChanged += CurrentBluetoothConnection_OnStateChanged;
            App.CurrentBluetoothConnection.OnRecived += CurrentBluetoothConnection_OnRecived;
            App.CurrentBluetoothConnection.OnError += CurrentBluetoothConnection_OnError;
        }
    }
    
    ...
    
    private void TransmitCurrentDigit()
    {
        DigitViewModel model = (DigitViewModel)BindingContext;
        if (model != null && !model.Reciving)
        {
            App.CurrentBluetoothConnection.Transmit(new Memory<byte>(new byte[] { model.Digit }));
        }
    }
    
    ...
```

### AndroidManifest.xml

Don't forget to add the next lines to the your {AplicationName}.Android/Properties/AndroidManifest.xml file:
```XML
<manifest xmlns:android="http://schemas.android.com/apk/res/android" 
          android:versionCode="1" android:versionName="1.0" 
          package="com.companyname.plugin.bluetoothclassic">
    ...
    <uses-feature android:name="android.hardware.bluetooth" />
    <uses-permission android:name="android.permission.BLUETOOTH" />
    <uses-permission android:name="android.permission.BLUETOOTH_ADMIN" />
    <uses-permission android:name="android.permission.ACCESS_COARSE_LOCATION" />
    <uses-permission android:name="android.permission.ACCESS_FINE_LOCATION" />
    ...
</manifest>
```

## Examples
The examples folder contains working example(s) that demonstrates how to use this package in practice.
* Retry example demonstractes how to work with a `IBluetoothConnection`
* Digit example demonstrates how to work with a `IBluetoothManagedConnection` one
