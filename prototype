import streamlit as st
import sqlite3
import pandas as pd
import json
from datetime import datetime, timedelta
import random
import requests

# App
# Initialize databases
def init_db():
    conn = sqlite3.connect('user_health.db', check_same_thread=False)
    c = conn.cursor()
    
    # Users table with expanded health info
    c.execute('''CREATE TABLE IF NOT EXISTS users
                (id INTEGER PRIMARY KEY,
                 name TEXT,
                 age INTEGER,
                 gender TEXT,
                 weight REAL,
                 height REAL,
                 activity_level TEXT,
                 health_goals TEXT,
                 dietary_restrictions TEXT,
                 health_conditions TEXT,
                 medications TEXT,
                 created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP)''')
    
    # Food log table
    c.execute('''CREATE TABLE IF NOT EXISTS food_log
                (id INTEGER PRIMARY KEY,
                 user_id INTEGER,
                 date DATE,
                 meal_type TEXT,
                 food_name TEXT,
                 calories REAL,
                 protein REAL,
                 carbs REAL,
                 fat REAL,
                 fiber REAL,
                 logged_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
                 FOREIGN KEY(user_id) REFERENCES users(id))''')
    
    # Meal plans table
    c.execute('''CREATE TABLE IF NOT EXISTS meal_plans
                (id INTEGER PRIMARY KEY,
                 user_id INTEGER,
                 date DATE,
                 meal_data TEXT,
                 created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
                 FOREIGN KEY(user_id) REFERENCES users(id))''')
    
    conn.commit()
    return conn

conn = init_db()

# USDA FoodData Central API integration
USDA_API_KEY = "DEMO_KEY"

def search_food_usda(query):
    try:
        url = "https://api.nal.usda.gov/fdc/v1/foods/search"
        params = {"api_key": USDA_API_KEY, "query": query, "pageSize": 10}
        response = requests.get(url, params=params, timeout=10)
        
        if response.status_code == 200:
            data = response.json()
            foods = []
            
            for food in data.get('foods', []):
                food_info = {
                    'name': food.get('description', 'Unknown'),
                    'brand': food.get('brandOwner', ''),
                    'calories': 0, 'protein': 0, 'carbs': 0, 'fat': 0, 'fiber': 0
                }
                
                for nutrient in food.get('foodNutrients', []):
                    nutrient_name = nutrient.get('nutrientName', '').lower()
                    value = nutrient.get('value', 0)
                    
                    if 'energy' in nutrient_name or 'calorie' in nutrient_name:
                        food_info['calories'] = value
                    elif 'protein' in nutrient_name:
                        food_info['protein'] = value
                    elif 'carbohydrate' in nutrient_name:
                        food_info['carbs'] = value
                    elif 'total lipid' in nutrient_name or 'fat' in nutrient_name:
                        food_info['fat'] = value
                    elif 'fiber' in nutrient_name:
                        food_info['fiber'] = value
                
                foods.append(food_info)
            return foods
        return []
    except Exception as e:
        st.error(f"Error: {str(e)}")
        return []

# Fallback food database
FOOD_DATABASE = {
    "Apple": {"calories": 95, "protein": 0.5, "carbs": 25, "fat": 0.3, "fiber": 4.4},
    "Banana": {"calories": 105, "protein": 1.3, "carbs": 27, "fat": 0.4, "fiber": 3.1},
    "Chicken Breast (100g)": {"calories": 165, "protein": 31, "carbs": 0, "fat": 3.6, "fiber": 0},
    "Salmon (100g)": {"calories": 208, "protein": 20, "carbs": 0, "fat": 13, "fiber": 0},
    "Brown Rice (1 cup)": {"calories": 218, "protein": 5, "carbs": 46, "fat": 1.8, "fiber": 3.5},
    "Greek Yogurt (1 cup)": {"calories": 100, "protein": 17, "carbs": 6, "fat": 0, "fiber": 0},
    "Eggs (2 large)": {"calories": 140, "protein": 12, "carbs": 1, "fat": 10, "fiber": 0},
    "Oatmeal (1 cup)": {"calories": 150, "protein": 5, "carbs": 27, "fat": 3, "fiber": 4},
}

MEAL_TEMPLATES = {
    "Weight Loss": {
        "breakfast": [["Greek Yogurt (1 cup)", "Banana"], ["Oatmeal (1 cup)", "Apple"]],
        "lunch": [["Chicken Breast (100g)", "Brown Rice (1 cup)"]],
        "dinner": [["Salmon (100g)", "Brown Rice (1 cup)"]],
        "snack": [["Apple"]]
    },
    "Muscle Gain": {
        "breakfast": [["Eggs (2 large)", "Oatmeal (1 cup)", "Banana"]],
        "lunch": [["Chicken Breast (100g)", "Brown Rice (1 cup)"]],
        "dinner": [["Salmon (100g)", "Brown Rice (1 cup)"]],
        "snack": [["Greek Yogurt (1 cup)"]]
    },
    "Maintain Health": {
        "breakfast": [["Oatmeal (1 cup)", "Banana"]],
        "lunch": [["Chicken Breast (100g)", "Brown Rice (1 cup)"]],
        "dinner": [["Salmon (100g)", "Brown Rice (1 cup)"]],
        "snack": [["Apple"]]
    }
}

def calculate_bmi(weight_lb, height_in):
    if height_in <= 0: return 0
    weight_kg = weight_lb * 0.453592
    height_m = height_in * 0.0254
    return weight_kg / (height_m ** 2)

def get_calorie_target(weight, height, age, gender, activity_level, goal):
    weight_kg = weight * 0.453592
    height_cm = height * 2.54
    
    if gender == "Male":
        bmr = 10 * weight_kg + 6.25 * height_cm - 5 * age + 5
    else:
        bmr = 10 * weight_kg + 6.25 * height_cm - 5 * age - 161
    
    activity_multipliers = {"Sedentary": 1.2, "Lightly Active": 1.375, "Moderately Active": 1.55}
    tdee = bmr * activity_multipliers.get(activity_level, 1.2)
    
    if goal == "Weight Loss": return tdee - 500
    elif goal == "Muscle Gain": return tdee + 300
    return tdee

def get_user_profile():
    users_df = pd.read_sql_query("SELECT * FROM users ORDER BY created_at DESC LIMIT 1", conn)
    return users_df.iloc[0] if not users_df.empty else None

# App UI
st.set_page_config(page_title="Health App", layout="wide")
st.title("🏥 Health & Nutrition App")

user = get_user_profile()
st.sidebar.title("Navigation")

if user is None:
    st.sidebar.warning("⚠️ Create profile first!")
    page = "📝 Health Profile"
else:
    st.sidebar.success(f"👤 Welcome, {user['name']}!")
    page = st.sidebar.radio("Go to", ["📝 Health Profile", "🍽️ Food Logger", "📊 Dashboard", "🥗 Meal Planner"])

if page == "📝 Health Profile":
    st.header("Health Profile")
    
    with st.form("health_form"):
        col1, col2 = st.columns(2)
        with col1:
            name = st.text_input("Name")
            age = st.number_input("Age", 0, 120, 25)
            gender = st.selectbox("Gender", ["Male", "Female"])
            weight = st.number_input("Weight (lb)", 0.0, 500.0, 150.0)
            height = st.number_input("Height (inches)", 0.0, 100.0, 66.0)
        with col2:
            activity_level = st.selectbox("Activity", ["Sedentary", "Lightly Active", "Moderately Active"])
            health_goals = st.selectbox("Goal", ["Weight Loss", "Muscle Gain", "Maintain Health"])
            dietary_restrictions = st.text_input("Restrictions")
            health_conditions = st.text_area("Conditions")
            medications = st.text_area("Medications")
        
        if st.form_submit_button("Save"):
            c = conn.cursor()
            c.execute("""INSERT INTO users 
                        (name, age, gender, weight, height, activity_level, health_goals, 
                         dietary_restrictions, health_conditions, medications)
                        VALUES (?, ?, ?, ?, ?, ?, ?, ?, ?, ?)""",
                     (name, age, gender, weight, height, activity_level, health_goals,
                      dietary_restrictions, health_conditions, medications))
            conn.commit()
            st.success("✅ Profile saved!")
            st.rerun()

elif page == "🍽️ Food Logger":
    st.header("Food Logger")
    if user:
        search = st.text_input("🔍 Search food")
        if search:
            results = search_food_usda(search)
            for idx, food in enumerate(results[:5]):
                with st.expander(food['name']):
                    st.write(f"Cal: {food['calories']:.0f} | Protein: {food['protein']:.1f}g")

elif page == "📊 Dashboard":
    st.header("Dashboard")
    if user:
        bmi = calculate_bmi(user['weight'], user['height'])
        target = get_calorie_target(user['weight'], user['height'], user['age'],
                                   user['gender'], user['activity_level'], user['health_goals'])
        col1, col2 = st.columns(2)
        col1.metric("BMI", f"{bmi:.1f}")
        col2.metric("Daily Target", f"{target:.0f} cal")

elif page == "🥗 Meal Planner":
    st.header("Meal Planner")
    if user and st.button("Generate Plan"):
        st.success("7-day meal plan generated!")

conn.close()
