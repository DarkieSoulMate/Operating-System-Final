AWS PROJECT FINAL ASSIGNMENT

LINKS:

S3: https://ap-south-1.console.aws.amazon.com/s3/buckets/kozimsbucket?region=ap-south-1&bucketType=general&tab=properties
EC2: https://ap-south-1.console.aws.amazon.com/ec2/home?region=ap-south-1#InstanceDetails:instanceId=i-0538daf2e7aacc9a9
RDS: https://ap-south-1.console.aws.amazon.com/rds/home?region=ap-south-1#database:id=dbkozim;is-cluster=false;tab=connectivity

HOW TO RUN APPLICATION:
- If not enabled, turn on EC2 and RDS resources
- Connect to EC2 Machine using ```ssh -i "kozimkey.pem" ubuntu@ec2-3-110-164-106.ap-south-1.compute.amazonaws.com``` command in terminal
- type ```source venv/bin/activate``` and head to webapp_kozim directory: ```cd webapp_kozim```
- run ```python app.py```
- open this link http://kozimsbucket.s3-website.ap-south-1.amazonaws.com/

Python code: 
```
from flask import Flask, request, jsonify
import psycopg2
from flask_cors import CORS

app = Flask(__name__)
CORS(app)

DB_CONFIG = {
    'dbname': 'postgres',
    'user': 'postgres',
    'password': 'postgres',
    'host': 'dbkozim.ct6ei6agkus4.ap-south-1.rds.amazonaws.com',
}

def get_db_connection():
    return psycopg2.connect(**DB_CONFIG)

@app.route('/movies')
def index():
    conn = get_db_connection()
    cur = conn.cursor()
    cur.execute('SELECT id, title, type, genres, releaseYear, imdbId, imdbAverageRating, imdbNumVotes, availableCountries FROM movies ORDER BY id;')
    movies = cur.fetchall()
    cur.close()
    conn.close()
    return jsonify(movies)

@app.route('/add', methods=['POST'])
def add_movie():
    data = (
        request.form['title'],
        request.form['type'],
        request.form['genres'],
        int(request.form['releaseYear']),
        request.form['imdbId'],
        float(request.form['imdbAverageRating']),
        int(request.form['imdbNumVotes']),
        request.form['availableCountries'] or None
    )

    conn = get_db_connection()
    cur = conn.cursor()
    cur.execute('''
        INSERT INTO movies (title, type, genres, releaseYear, imdbId, imdbAverageRating, imdbNumVotes, availableCountries)
        VALUES (%s, %s, %s, %s, %s, %s, %s, %s);
    ''', data)
    conn.commit()
    cur.close()
    conn.close()
    return jsonify({'message': 'Movie added successfully'})

@app.route('/delete', methods=['POST'])
def delete_movie(movie_id):
    movie_id = request.json.get('id')
    conn = get_db_connection()
    cur = conn.cursor()
    cur.execute('DELETE FROM movies WHERE id = %s;', (movie_id,))
    conn.commit()
    cur.close()
    conn.close()
    return jsonify({'message': 'Movie with id {movie_id} deleted'})

if __name__ == '__main__':
    app.run(host="0.0.0.0", port=8000)
```


HTML webpage code:
```
<!DOCTYPE html>
<html lang="en">
<head>
  <meta charset="UTF-8" />
  <meta name="viewport" content="width=device-width, initial-scale=1.0"/>
  <title>Movies List</title>
  <style>
    body { font-family: Arial, sans-serif; padding: 20px; }
    table { border-collapse: collapse; width: 100%; margin-top: 20px; }
    th, td { border: 1px solid #ccc; padding: 8px; text-align: left; }
    th { background-color: #f2f2f2; }
    input, button { margin: 5px; padding: 5px; }
  </style>
</head>
<body>
  <h1>Movies Database</h1>

  <div>
    <h3>Add Movie</h3>
    <input type="text" id="title" placeholder="Title" />
    <input type="text" id="type" placeholder="Type" />
    <input type="text" id="genres" placeholder="Genres" />
    <input type="number" id="releaseyear" placeholder="Release Year" />
    <input type="text" id="imdbid" placeholder="IMDb ID" />
    <input type="number" step="0.1" id="imdbaveragerating" placeholder="IMDb Rating" />
    <input type="number" id="imdbnumvotes" placeholder="IMDb Votes" />
    <input type="text" id="availablecountries" placeholder="Countries" />
    <button onclick="addMovie()">Add Movie</button>
  </div>

  <table id="moviesTable">
    <thead>
      <tr>
        <th>ID</th><th>Title</th><th>Type</th><th>Genres</th><th>Year</th><th>IMDb ID</th>
        <th>Rating</th><th>Votes</th><th>Countries</th><th>Action</th>
      </tr>
    </thead>
    <tbody></tbody>
  </table>

  <script>
    const apiBase = "http://3.110.164.106:8000";

    async function loadMovies() {
      const res = await fetch(`${apiBase}/movies`);
      const movies = await res.json();
      const tbody = document.querySelector("#moviesTable tbody");
      tbody.innerHTML = "";

      movies.forEach(movie => {
        const row = document.createElement("tr");
        row.innerHTML = `
          <td>${movie[0]}</td>
          <td>${movie[1]}</td>
          <td>${movie[2]}</td>
          <td>${movie[3]}</td>
          <td>${movie[4]}</td>
          <td>${movie[5]}</td>
          <td>${movie[6]}</td>
          <td>${movie[7]}</td>
          <td>${movie[8] ?? ""}</td>
          <td><button onclick="deleteMovie(${movie[0]})">Delete</button></td>
        `;
        tbody.appendChild(row);
      });
    }

async function addMovie() {
  const formData = new FormData();
  formData.append("title", document.getElementById("title").value);
  formData.append("type", document.getElementById("type").value);
  formData.append("genres", document.getElementById("genres").value);
  formData.append("releaseYear", document.getElementById("releaseyear").value);
  formData.append("imdbId", document.getElementById("imdbid").value);
  formData.append("imdbAverageRating", document.getElementById("imdbaveragerating").value);
  formData.append("imdbNumVotes", document.getElementById("imdbnumvotes").value);
  formData.append("availableCountries", document.getElementById("availablecountries").value);

  const res = await fetch(`${apiBase}/add`, {
    method: "POST",
    body: formData
  });

  if (res.ok) {
    alert("Movie added!");
    loadMovies();
  } else {
    alert("Failed to add movie.");
  }
}

    async function deleteMovie(id) {
      const res = await fetch(`${apiBase}/delete`, {
        method: "POST",
        headers: { "Content-Type": "application/json" },
        body: JSON.stringify({ id })
      });

      if (res.ok) {
        alert("Movie deleted.");
        loadMovies();
      } else {
        alert("Failed to delete.");
      }
    }

    loadMovies();
  </script>
</body>
</html>
```
