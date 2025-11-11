import { useState } from "react";
import Papa from "papaparse";
import { Bar } from "react-chartjs-2";
import {
  Chart as ChartJS,
  CategoryScale,
  LinearScale,
  BarElement,
  Title,
  Tooltip,
  Legend,
} from "chart.js";
import "./App.css";

ChartJS.register(CategoryScale, LinearScale, BarElement, Title, Tooltip, Legend);

export default function App() {
  const [data, setData] = useState([]);
  const [error, setError] = useState("");
  const [metrics, setMetrics] = useState(null);
  const [perPerson, setPerPerson] = useState({});

  const expectedHeaders = ["name", "date", "value"];

  function handleFileUpload(e) {
    const file = e.target.files[0];
    if (!file) return;

    Papa.parse(file, {
      header: true,
      skipEmptyLines: true,
      complete: (results) => validateCSV(results.data, results.meta.fields),
      error: (err) => setError("Error parsing CSV: " + err.message),
    });
  }

  function validateCSV(rows, headers) {
    setError("");
    // 1. Validate headers
    const missing = expectedHeaders.filter((h) => !headers.includes(h));
    if (missing.length > 0) {
      setError(`Invalid CSV headers. Missing: ${missing.join(", ")}`);
      setData([]);
      return;
    }

    // 2. Validate data types
    const invalid = rows.find(
      (r) =>
        !r.name ||
        isNaN(Number(r.value)) ||
        isNaN(new Date(r.date).getTime())
    );
    if (invalid) {
      setError("Invalid data type detected (check 'value' or 'date' columns).");
      setData([]);
      return;
    }

    // All good
    setData(rows);
    computeMetrics(rows);
  }

  function computeMetrics(rows) {
    if (!rows.length) return;

    const numericValues = rows.map((r) => Number(r.value));
    const overall = {
      average: (numericValues.reduce((a, b) => a + b, 0) / numericValues.length).toFixed(2),
      min: Math.min(...numericValues),
      max: Math.max(...numericValues),
    };

    // Group by person
    const grouped = {};
    rows.forEach((r) => {
      const val = Number(r.value);
      if (!grouped[r.name]) grouped[r.name] = [];
      grouped[r.name].push(val);
    });

    const per = {};
    for (const [name, vals] of Object.entries(grouped)) {
      per[name] = {
        average: (vals.reduce((a, b) => a + b, 0) / vals.length).toFixed(2),
        min: Math.min(...vals),
        max: Math.max(...vals),
      };
    }

    setMetrics(overall);
    setPerPerson(per);
  }

  const perPersonChartData = {
    labels: Object.keys(perPerson),
    datasets: [
      {
        label: "Average",
        data: Object.values(perPerson).map((p) => p.average),
        backgroundColor: "rgba(75,192,192,0.6)",
      },
    ],
  };

  return (
    <div className="App">
      <h1>📊 CSV Data Visualizer</h1>
      <input type="file" accept=".csv" onChange={handleFileUpload} />

      {error && <p className="error">{error}</p>}

      {data.length > 0 && (
        <>
          <h2>Overall Metrics</h2>
          <ul>
            <li>Average: {metrics.average}</li>
            <li>Min: {metrics.min}</li>
            <li>Max: {metrics.max}</li>
          </ul>

          <h2>Per-Person Metrics</h2>
          <table>
            <thead>
              <tr>
                <th>Name</th>
                <th>Average</th>
                <th>Min</th>
                <th>Max</th>
              </tr>
            </thead>
            <tbody>
              {Object.entries(perPerson).map(([name, m]) => (
                <tr key={name}>
                  <td>{name}</td>
                  <td>{m.average}</td>
                  <td>{m.min}</td>
                  <td>{m.max}</td>
                </tr>
              ))}
            </tbody>
          </table>

          <h2>Chart: Average Value per Person</h2>
          <div style={{ width: "600px", margin: "auto" }}>
            <Bar data={perPersonChartData} />
          </div>
        </>
      )}
    </div>
  );
}

 src/App.css
.App {
  text-align: center;
  font-family: Arial, sans-serif;
  padding: 2rem;
}

.error {
  color: red;
  font-weight: bold;
}

table {
  border-collapse: collapse;
  margin: 1rem auto;
  width: 60%;
}

th, td {
  border: 1px solid #ccc;
  padding: 8px 12px;
}

th {
  background: #f2f2f2;
}

input[type="file"] {
  margin: 1rem 0;
}
