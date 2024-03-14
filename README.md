# Zavrsni-rad
using System;
using System.Collections.Generic;
using System.Globalization;
using System.IO;
using System.Linq;
using System.Net;
using System.Text;
using System.Threading.Tasks;
using System.Windows;
using System.Windows.Controls;
using System.Windows.Data;
using System.Windows.Documents;
using System.Windows.Input;
using System.Windows.Media;
using System.Windows.Media.Imaging;
using System.Windows.Navigation;
using System.Windows.Shapes;
using Newtonsoft.Json.Linq;
using System.Net.NetworkInformation;
using System.Globalization;

namespace Zavrsni_rad
{
    /// <summary>
    /// Interaction logic for MainWindow.xaml
    /// </summary>
    public partial class MainWindow : Window
    {
        public MainWindow()
        {
            InitializeComponent();
            
        }

        public void OnLoad(object sender, RoutedEventArgs e)
        {
            GetConversionRates();
        }

        List<CurrencyInfo> currencyList = new List<CurrencyInfo>();

        public float GetFloat(JToken value)
        {
            float result;

            // Try parsing in the current culture
            if (!float.TryParse(value.ToString(), System.Globalization.NumberStyles.Any, CultureInfo.CurrentCulture, out result) &&
                // Then try in US english
                !float.TryParse(value.ToString(), System.Globalization.NumberStyles.Any, CultureInfo.GetCultureInfo("en-US"), out result) &&
                // Then in neutral language
                !float.TryParse(value.ToString(), System.Globalization.NumberStyles.Any, CultureInfo.InvariantCulture, out result))
            {
                result = 1;
            }
            return result;
        }
        public void GetConversionRates()
        {
            try
            {
                List<string> currencies = new List<string>();
                var url = "https://api.hnb.hr/tecajn-eur/v3";
                JArray parsedResponse;
                var httpRequest = (HttpWebRequest)WebRequest.Create(url);
                httpRequest.Method = "GET";
                var httpResponse = (HttpWebResponse)httpRequest.GetResponse();


                using (var streamReader = new StreamReader(httpResponse.GetResponseStream()))
                {
                    var responseJson = streamReader.ReadToEnd();
                    parsedResponse = JArray.Parse(responseJson);
                }
                foreach (var currency in parsedResponse)
                {
                    currencyList.Add(new CurrencyInfo((string)currency["valuta"], GetFloat(currency["kupovni_tecaj"]), (float)currency["prodajni_tecaj"]));
                    currencies.Add(currency["valuta"].ToString());
                }
                currencies.Add("EUR");
                currencyList.Add(new CurrencyInfo(("EUR"), 1, 1));
                currencies.Sort();
                foreach (string cur in currencies)
                {
                    CboFrom.Items.Add(cur);
                    CboTo.Items.Add(cur);
                }

                //Update label info
                //lblUpdateInfo.Content = $"Conversion rates updated {DateTime.Now.ToString("G")}";
            }
            catch (WebException)
            {

                MessageBox.Show("Failed to load currency rate. Using outdated rates. Check your internet connection. ");
            }
    }

        private void ButtonConvert_Click(object sender, RoutedEventArgs e)
        {
            float amount;
            string toCurrency, fromCurrency;
            try
            {
                amount = float.Parse(TextBoxAmount.Text);
                toCurrency = (string)CboTo.SelectedItem;
                fromCurrency = (string)CboFrom.SelectedItem;

                if (FindConversionRate(fromCurrency, toCurrency) > 0)
                {
                    TextBoxTotal.Text = $"{amount} {fromCurrency} is {Math.Round(FindConversionRate(fromCurrency, toCurrency) * amount, 2)} {toCurrency}";
                }
            }
            catch (FormatException)
            {

                MessageBox.Show("Entered amount is invalid. Please check it and retry.");
            }
        }

        public float FindConversionRate(string fromCurrency, string toCurrency)
        {

            if (fromCurrency != "EUR")
            {
                float fromBuyingRate;
                float toBuyingRate;
                if (fromCurrency == null || toCurrency == null)
                {
                    MessageBox.Show("Please select both From and To currency!");
                    return 0;
                }
                else
                {
                    fromBuyingRate = currencyList.Find(x => x.Currency == fromCurrency).BuyingRate;
                    toBuyingRate = currencyList.Find(x => x.Currency == toCurrency).BuyingRate;

                }

                if (fromBuyingRate > toBuyingRate)
                {
                    return ((1 / fromBuyingRate)) * toBuyingRate;
                }
                else if (fromBuyingRate < toBuyingRate)
                {
                    return ((1 / fromBuyingRate) * toBuyingRate);
                }
                else
                {
                    return 1;
                }
            }
            else //if fromCurrency was EUR just match it with proper conversion rate
            {
                return currencyList.Find(x => x.Currency == toCurrency).BuyingRate;
            }
        }
    }
}
