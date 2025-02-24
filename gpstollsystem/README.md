UNIQUENESS OF OUR PROJECT

Our GPS toll-based system, developed using Python, stands out for its seamless integration of real-time GPS data to automate toll collection processes. This innovative solution eliminates the need for traditional toll booths, reducing traffic congestion and enhancing the efficiency of toll roads. By leveraging Python's robust libraries for geolocation and data processing, our system accurately tracks vehicle movements and calculates toll charges based on distance traveled and specific road usage. The use of a cloud-based backend ensures scalability and real-time data processing, providing an unmatched user experience. Additionally, the implementation of secure payment gateways and user-friendly interfaces further distinguishes our project, offering a comprehensive, modern, and efficient solution to toll management.

IMPLEMENTATION STEPS

import simpy
import geopandas as gpd
from geopy.distance import geodesic
from shapely.geometry import Point, LineString
import matplotlib.pyplot as plt

# Define the new toll booth location
toll_booth_location = (9.4981, 76.3388)  # Alappuzha, Kerala

# Define vehicle locations with (latitude, longitude), initial account balances, base toll amounts, and names
vehicle_data = [
    {'type': 'Truck', 'name': 'Kottayam', 'location': (9.5916, 76.5221), 'balance': 10000.0, 'destination': (9.9312, 76.2673), 'base_toll': 20.0, 'start_time': 6 * 60},
    {'type': 'Car', 'name': 'Changanassery', 'location': (9.4420, 76.5363), 'balance': 5000.0, 'destination': (9.9833, 76.6150), 'base_toll': 5.0, 'start_time': 11 * 60},
    {'type': 'Bus', 'name': 'Puthupally', 'location': (9.5674, 76.6310), 'balance': 7500.0, 'destination': (9.9815, 76.5735), 'base_toll': 10.0, 'start_time': 8 * 60}
]

# Define a GeoDataFrame for vehicle starting points, toll booth, and destinations
gdf_points = gpd.GeoDataFrame(
    {
        'id': ['Toll Booth', 'Truck Start', 'Car Start', 'Bus Start', 'Truck End', 'Car End', 'Bus End'],
        'geometry': gpd.points_from_xy(
            [toll_booth_location[1], vehicle_data[0]['location'][1], vehicle_data[1]['location'][1], vehicle_data[2]['location'][1],
             vehicle_data[0]['destination'][1], vehicle_data[1]['destination'][1], vehicle_data[2]['destination'][1]],
            [toll_booth_location[0], vehicle_data[0]['location'][0], vehicle_data[1]['location'][0], vehicle_data[2]['location'][0],
             vehicle_data[0]['destination'][0], vehicle_data[1]['destination'][0], vehicle_data[2]['destination'][0]]
        )
    }
)

# Define a GeoDataFrame for routes from vehicles to the toll booth and to their destinations
gdf_routes = gpd.GeoDataFrame(
    {
        'id': ['Truck Route', 'Car Route', 'Bus Route'],
        'geometry': [
            LineString([Point(vehicle_data[0]['location'][1], vehicle_data[0]['location'][0]), Point(toll_booth_location[1], toll_booth_location[0]),
                        Point(vehicle_data[0]['destination'][1], vehicle_data[0]['destination'][0])]),
            LineString([Point(vehicle_data[1]['location'][1], vehicle_data[1]['location'][0]), Point(toll_booth_location[1], toll_booth_location[0]),
                        Point(vehicle_data[1]['destination'][1], vehicle_data[1]['destination'][0])]),
            LineString([Point(vehicle_data[2]['location'][1], vehicle_data[2]['location'][0]), Point(toll_booth_location[1], toll_booth_location[0]),
                        Point(vehicle_data[2]['destination'][1], vehicle_data[2]['destination'][0])])
        ]
    }
)

# Define congestion level that affects toll prices
congestion_level = 1.5  # 1.0 = normal, > 1.0 = high congestion, < 1.0 = low congestion

# Define dynamic toll pricing based on time of day and congestion
def get_dynamic_toll(base_toll, vehicle_type, time_of_day, congestion_level):
    # Peak hours are 8-10 AM and 5-7 PM
    if 8 <= time_of_day < 10 or 17 <= time_of_day < 19:
        peak_factor = 1.5
    else:
        peak_factor = 1.0

    # Different toll rates for different vehicle types
    vehicle_factor = 1.0
    if vehicle_type == 'Truck':
        vehicle_factor = 2.0
    elif vehicle_type == 'Bus':
        vehicle_factor = 1.5

    toll = base_toll * peak_factor * vehicle_factor * congestion_level

    # Increase toll by additional percentages during specific time intervals
    if 6.5 <= time_of_day < 7.5:
        toll *= 1.05
        print(f"Additional 5% toll applied for {vehicle_type} at {time_of_day:.2f} hours.")
    elif 8.5 <= time_of_day < 9.5:
        toll *= 1.08
        print(f"Additional 8% toll applied for {vehicle_type} at {time_of_day:.2f} hours.")
    elif 11.5 <= time_of_day < 12.5:
        toll *= 1.10
        print(f"Additional 10% toll applied for {vehicle_type} at {time_of_day:.2f} hours.")

    return toll

# Define a function to convert time in hours to HH:MM format
def convert_to_hhmm(hours):
    hh = int(hours)
    mm = int((hours - hh) * 60)
    return f"{hh:02}:{mm:02}"

# Define a function to simulate vehicles passing through a toll
def vehicle(env, vehicle_id, data):
    vehicle_type = data['type']
    name = data['name']
    location = data['location']
    balance = data['balance']
    base_toll = data['base_toll']
    destination = data['destination']
    start_time = data['start_time']

    yield env.timeout(start_time)  # Start the journey at the specified time
    start_time_hr = env.now / 60  # Convert start time to hours
    print(f"Time: {convert_to_hhmm(start_time_hr)} - {vehicle_type} from {name} ({location}) starts the journey with a balance of {balance:.2f}.")

    # Simulate arrival time based on distance and an average speed of 50 km/h
    distance_to_toll = geodesic(location, toll_booth_location).km
    travel_time = distance_to_toll / 50 * 60  # convert hours to minutes
    yield env.timeout(travel_time)

    time_of_day = (env.now / 60) % 24  # Get the current time in hours
    total_toll = get_dynamic_toll(base_toll, vehicle_type, time_of_day, congestion_level)

    if balance >= total_toll:
        balance -= total_toll
        print(f"Time: {convert_to_hhmm(env.now / 60)} - {vehicle_type} from {name} ({location}) reached the toll booth, incurred a toll of {total_toll:.2f}, and has a remaining balance of {balance:.2f}.")
    else:
        print(f"Time: {convert_to_hhmm(env.now / 60)} - {vehicle_type} from {name} ({location}) reached the toll booth but does not have enough balance to pay the toll.")

    # Simulate remaining travel to destination
    distance_to_destination = geodesic(toll_booth_location, destination).km
    travel_time_to_destination = distance_to_destination / 50 * 60
    yield env.timeout(travel_time_to_destination)
    end_time_hr = env.now / 60  # Convert end time to hours
    print(f"Time: {convert_to_hhmm(end_time_hr)} - {vehicle_type} has reached its destination at {destination}.")

# Define a simulation environment
env = simpy.Environment()

# Create processes for each vehicle
for i, data in enumerate(vehicle_data, start=1):
    env.process(vehicle(env, i, data))

# Run the simulation
env.run()

# Plot the locations and routes on a map with different colors for each vehicle's route
fig, ax = plt.subplots(1, 1, figsize=(10, 10))
gdf_points.plot(ax=ax, color='blue', marker='o', markersize=50, label='Locations')
gdf_routes[gdf_routes['id'] == 'Truck Route'].plot(ax=ax, color='orange', linewidth=2, label='Truck Route')
gdf_routes[gdf_routes['id'] == 'Car Route'].plot(ax=ax, color='green', linewidth=2, label='Car Route')
gdf_routes[gdf_routes['id'] == 'Bus Route'].plot(ax=ax, color='purple', linewidth=2, label='Bus Route')

# Annotate the points with vehicle type and names
for x, y, label in zip(gdf_points.geometry.x, gdf_points.geometry.y, gdf_points.id):
    ax.text(x, y, label, fontsize=12, ha='right')

plt.title("Vehicle Routes and Toll Booth in Kerala")
plt.legend()
plt.show()
