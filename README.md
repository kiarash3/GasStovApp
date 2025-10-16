# GasStovApp
using System;
using System.ComponentModel;

namespace GasStoveApp
{
    class Program
    {
        static void Main(string[] args)
        {
            Console.WriteLine("Press any key to start the gas stove...");
            Console.ReadKey();

            IDevice device = new GasStoveDevice();

            device.RunDevice();

            Console.ReadKey();
        }
    }

    public interface IDevice
    {
        double WarningTemperatureLevel { get; }
        double EmergencyTemperatureLevel { get; }
        
        void RunDevice();
        void HandleEmergency();
    }

    public class GasStoveDevice : IDevice
    {
        const double Warning_Level = 250; // Min safe temperature
        const double Emergency_Level = 650; // Max safe temperature

        public double WarningTemperatureLevel => Warning_Level;
        public double EmergencyTemperatureLevel => Emergency_Level;

        public void HandleEmergency()
        {
            Console.WriteLine();
            Console.WriteLine("🚨 Gas leak risk detected! Shutting off system...");
            ShutDownDevice();
            Console.WriteLine();
        }

        private void ShutDownDevice()
        {
            Console.WriteLine("💤 The stove has been turned off (Safety mode activated).");
        }

        public void RunDevice()
        {
            Console.WriteLine("🔥 Gas stove is running...");

            IFlameController flameController = new FlameController();
            IHeatSensor heatSensor = new HeatSensor(Warning_Level, Emergency_Level);
            IThermostat thermostat = new Thermostat(this, heatSensor, flameController);

            thermostat.RunThermostat();
        }
    }

    // 🔥 کنترل شعله
    public interface IFlameController
    {
        void IncreaseGasFlow();
        void ReduceGasFlow();
        void ShutOffGas();
    }

    public class FlameController : IFlameController
    {
        public void IncreaseGasFlow()
        {
            Console.WriteLine();
            Console.WriteLine("🔥 Gas flow increased to maintain flame stability.");
        }

        public void ReduceGasFlow()
        {
            Console.WriteLine();
            Console.WriteLine("❄️ Gas flow reduced to lower the temperature.");
        }

        public void ShutOffGas()
        {
            Console.WriteLine();
            Console.WriteLine("🚫 Gas supply completely shut off!");
        }
    }

    // 🌡️ ترموستات
    public interface IThermostat
    {
        void RunThermostat();
    }

    public class Thermostat : IThermostat
    {
        private readonly IFlameController _flameController;
        private readonly IHeatSensor _heatSensor;
        private readonly IDevice _device;

        public Thermostat(IDevice device, IHeatSensor heatSensor, IFlameController flameController)
        {
            _device = device;
            _heatSensor = heatSensor;
            _flameController = flameController;
        }

        public void RunThermostat()
        {
            Console.WriteLine("🌡️ Thermostat is monitoring temperature...");
            WireUpEvents();
            _heatSensor.RunHeatSensor();
        }

        private void WireUpEvents()
        {
            _heatSensor.TemperatureReachesWarningLevelEventHandler += OnWarningLevel;
            _heatSensor.TemperatureFallsBelowWarningLevelEventHandler += OnCoolDown;
            _heatSensor.TemperatureReachesEmergencyLevelEventHandler += OnEmergencyLevel;
        }

        private void OnWarningLevel(object sender, TemperatureEventArgs e)
        {
            Console.ForegroundColor = ConsoleColor.Yellow;
            Console.WriteLine($"⚠️ Warning: Temperature reached {e.Temperature}°C! Reducing gas flow...");
            _flameController.ReduceGasFlow();
            Console.ResetColor();
        }

        private void OnCoolDown(object sender, TemperatureEventArgs e)
        {
            Console.ForegroundColor = ConsoleColor.Cyan;
            Console.WriteLine($"ℹ️ Info: Temperature dropped to {e.Temperature}°C. Increasing gas flow...");
            _flameController.IncreaseGasFlow();
            Console.ResetColor();
        }

        private void OnEmergencyLevel(object sender, TemperatureEventArgs e)
        {
            Console.ForegroundColor = ConsoleColor.Red;
            Console.WriteLine($"🚨 Emergency! Temperature reached {e.Temperature}°C!");
            _flameController.ShutOffGas();
            _device.HandleEmergency();
            Console.ResetColor();
        }
    }

    // 🌡️ حسگر دما
    public interface IHeatSensor
    {
        event EventHandler<TemperatureEventArgs> TemperatureReachesWarningLevelEventHandler;
        event EventHandler<TemperatureEventArgs> TemperatureReachesEmergencyLevelEventHandler;
        event EventHandler<TemperatureEventArgs> TemperatureFallsBelowWarningLevelEventHandler;
        void RunHeatSensor();
    }

    public class HeatSensor : IHeatSensor
    {
        private readonly double _warningLevel;
        private readonly double _emergencyLevel;
        private bool _hasReachedWarningTemperature = false;

        protected EventHandlerList _listEventDelegates = new EventHandlerList();

        static readonly object _temperatureReachesWarningLevelKey = new object();
        static readonly object _temperatureFallsBelowWarningLevelKey = new object();
        static readonly object _temperatureReachesEmergencyLevelKey = new object();

        private double[] _temperatureData = null;

        public HeatSensor(double warningLevel, double emergencyLevel)
        {
            _warningLevel = warningLevel;
            _emergencyLevel = emergencyLevel;

            SeedData();
        }

        private void SeedData()
        {
            _temperatureData = new double[] {180, 220, 240, 260, 300, 400, 480, 680, 640, 420, 300, 200};
        }

        private void MonitorTemperature()
        {
            foreach (double temp in _temperatureData)
            {
                Console.ResetColor();
                Console.WriteLine($"DateTime: {DateTime.Now}, Temperature: {temp}°C");

                if (temp >= _emergencyLevel)
                    OnTemperatureReachesEmergencyLevel(new TemperatureEventArgs { Temperature = temp, CurrentDateTime = DateTime.Now });
                else if (temp >= _warningLevel)
                {
                    _hasReachedWarningTemperature = true;
                    OnTemperatureReachesWarningLevel(new TemperatureEventArgs { Temperature = temp, CurrentDateTime = DateTime.Now });
                }
                else if (temp < _warningLevel && _hasReachedWarningTemperature)
                {
                    _hasReachedWarningTemperature = false;
                    OnTemperatureFallsBelowWarningLevel(new TemperatureEventArgs { Temperature = temp, CurrentDateTime = DateTime.Now });
                }

                System.Threading.Thread.Sleep(1000);
            }
        }

        protected void OnTemperatureReachesWarningLevel(TemperatureEventArgs e)
        {
            ((EventHandler<TemperatureEventArgs>)_listEventDelegates[_temperatureReachesWarningLevelKey])?.Invoke(this, e);
        }

        protected void OnTemperatureFallsBelowWarningLevel(TemperatureEventArgs e)
        {
            ((EventHandler<TemperatureEventArgs>)_listEventDelegates[_temperatureFallsBelowWarningLevelKey])?.Invoke(this, e);
        }

        protected void OnTemperatureReachesEmergencyLevel(TemperatureEventArgs e)
        {
            ((EventHandler<TemperatureEventArgs>)_listEventDelegates[_temperatureReachesEmergencyLevelKey])?.Invoke(this, e);
        }

        event EventHandler<TemperatureEventArgs> IHeatSensor.TemperatureReachesWarningLevelEventHandler
        {
            add => _listEventDelegates.AddHandler(_temperatureReachesWarningLevelKey, value);
            remove => _listEventDelegates.RemoveHandler(_temperatureReachesWarningLevelKey, value);
        }

        event EventHandler<TemperatureEventArgs> IHeatSensor.TemperatureFallsBelowWarningLevelEventHandler
        {
            add => _listEventDelegates.AddHandler(_temperatureFallsBelowWarningLevelKey, value);
            remove => _listEventDelegates.RemoveHandler(_temperatureFallsBelowWarningLevelKey, value);
        }

        event EventHandler<TemperatureEventArgs> IHeatSensor.TemperatureReachesEmergencyLevelEventHandler
        {
            add => _listEventDelegates.AddHandler(_temperatureReachesEmergencyLevelKey, value);
            remove => _listEventDelegates.RemoveHandler(_temperatureReachesEmergencyLevelKey, value);
        }

        public void RunHeatSensor()
        {
            Console.WriteLine("🔥 Heat sensor is active...");
            MonitorTemperature();
        }
    }

    // 📊 داده‌های دما
    public class TemperatureEventArgs : EventArgs
    {
        public double Temperature { get; set; }
        public DateTime CurrentDateTime { get; set; }
    }
}
