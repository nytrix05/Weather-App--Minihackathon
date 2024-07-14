import openmeteo_requests
import requests_cache
import requests
from kivy.app import App
from kivy.uix.screenmanager import ScreenManager, Screen
from kivy.uix.button import Button
from kivy.uix.label import Label
from kivy.uix.textinput import TextInput
from kivy.uix.gridlayout import GridLayout
from kivy.uix.floatlayout import FloatLayout
from kivy.uix.image import Image
from kivy.uix.widget import Widget

# Setup the Open-Meteo API client with caching
cache_session = requests_cache.CachedSession('.cache', expire_after=3600)


class MainScreen(Screen):
    def build(self):
        layout = GridLayout(cols=1, spacing=10, padding=10)
        layout.size_hint = (None, None)
        layout.width = 400
        layout.pos_hint = {"center_x": 0.5, "center_y": 0.5}

        label = Label(
            text="Bitte gebe deinen gewünschten Standort ein ?",
            size_hint=(1, None),
            height=50,
            halign='center'
        )
        label.bind(size=label.setter('text_size'))
        layout.add_widget(label)

        self.textfield = TextInput(
            size_hint=(1, None),
            height=50,
        )
        layout.add_widget(self.textfield)

        button = Button(
            text="Absenden",
            size_hint=(1, None),
            height=50,
        )
        button.bind(on_press=self.wettergo)
        layout.add_widget(button)

        self.add_widget(layout)
        return self

    def wettergo(self, instance):
        location = self.textfield.text
        self.manager.current = "wetter"
        self.manager.get_screen("wetter").update_weather(location)


class Wetter(Screen):
    def build(self):
        layout = FloatLayout()

        self.button = Button(
            background_normal="paint.png",
            background_down="paint.png",
            size_hint=(None, None),
            size=(100, 100),
            pos_hint={"x": 0, "y": 0.9}
        )
        self.button.bind(on_press=self.back_to_menu)
        layout.add_widget(self.button)

        central_layout = GridLayout(cols=3, spacing=10, padding=10, size_hint=(None, None), width=550)
        central_layout.pos_hint = {"center_x": 0.5, "center_y": 0.5}

        self.image = Image(
            source="wetterjo.png",
            size_hint=(None, None),
            size=(250, 250),
            pos_hint = {"center_x": 0.5, "center_y": 0.5}
        )
        central_layout.add_widget(self.image)

        self.weather_label = Label(
            text="Wetterdaten werden hier angezeigt.",
            size_hint=(1, None),
            pos_hint = {"center_x": 0.5, "center_y": 0.5},
            halign = "center"
            
        )
        self.weather_label.bind(size=self.weather_label.setter('text_size'))
        central_layout.add_widget(self.weather_label)

        layout.add_widget(central_layout)
        self.add_widget(layout)
        return self
    
    def back_to_menu(self, instance):
            self.manager.current = 'main'
            self.manager.get_screen('main')

            
    def update_weather(self, location):
        # Geocoding API call
        geocode_url = "https://api.opencagedata.com/geocode/v1/json"
        geocode_params = {
            "q": location,
            "key": "2241ecbdb58a4effb8458fde965032a8"  # Replace with your OpenCage API key
        }
        geocode_response = requests.get(geocode_url, params=geocode_params).json()

        if geocode_response['results']:
            latitude = geocode_response['results'][0]['geometry']['lat']
            longitude = geocode_response['results'][0]['geometry']['lng']
        else:
            self.weather_label.text = "Standort nicht gefunden."
            return

        weather_params = {
            "latitude": latitude,
            "longitude": longitude,
            "hourly": ["temperature_2m", "rain", "wind_speed_10m"],
        }

        response = openmeteo_requests.Client().weather_api("https://api.open-meteo.com/v1/forecast", params=weather_params)
        response = response[0]

        hourly = response.Hourly()
        temperature = hourly.Variables(0).ValuesAsNumpy()
        rain = hourly.Variables(1).ValuesAsNumpy()
        wind_speed = hourly.Variables(2).ValuesAsNumpy()

        # Update the label with weather data
        self.weather_label.text = (
            f"Wetter in {location}:\n"
            f"Temperatur: {temperature.mean():.1f}°C\n"
            f"Regen: {rain.sum():.0f} %\n"
            f"Windgeschwindigkeit: {wind_speed.mean():.1f} m/s"
        )
        
        


class MyApp(App):
    def build(self):
        sm = ScreenManager()
        main_screen = MainScreen(name='main')
        main_screen.build()
        sm.add_widget(main_screen)

        wetter_screen = Wetter(name='wetter')
        wetter_screen.build()
        sm.add_widget(wetter_screen)

        return sm


if __name__ == "__main__":
    MyApp().run()
