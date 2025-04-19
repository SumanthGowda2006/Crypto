<!DOCTYPE html>
<html lang="en">
<head>
  <meta charset="UTF-8" />
  <meta name="viewport" content="width=device-width, initial-scale=1.0"/>
  <title>Crypto Portfolio Tracker</title>
  <style>
    body {
      font-family: Arial, sans-serif;
      background-color: #f9f9f9;
      margin: 0;
      padding: 2rem;
    }
    h1 {
      text-align: center;
    }
    .container {
      max-width: 1000px;
      margin: auto;
    }
    .search-sort {
      display: flex;
      justify-content: space-between;
      align-items: center;
      margin-bottom: 1rem;
    }
    input[type="text"], input[type="number"] {
      padding: 0.5rem;
      margin: 0.3rem;
    }
    table {
      width: 100%;
      border-collapse: collapse;
      margin-bottom: 2rem;
    }
    th, td {
      padding: 0.75rem;
      border: 1px solid #ccc;
      text-align: center;
    }
    th.sortable:hover {
      cursor: pointer;
      background-color: #eee;
    }
    .alert-item {
      display: flex;
      justify-content: space-between;
      align-items: center;
      padding: 0.5rem 0;
    }
    button {
      padding: 0.4rem 0.8rem;
      cursor: pointer;
    }
  </style>
</head>
<body>
  <div class="container">
    <h1>Crypto Portfolio Tracker</h1>

    <div class="search-sort">
      <input type="text" id="searchInput" placeholder="Search for a coin..." oninput="renderTable()">
    </div>

    <h2>Top Coins</h2>
    <table id="coinTable">
      <thead>
        <tr>
          <th class="sortable" onclick="sortTable('name')">Name</th>
          <th class="sortable" onclick="sortTable('price')">Price (USD)</th>
          <th>Qty</th>
          <th>Add</th>
          <th>Set Alert</th>
        </tr>
      </thead>
      <tbody></tbody>
    </table>

    <h2>Your Portfolio</h2>
    <table id="portfolioTable">
      <thead>
        <tr>
          <th>Name</th>
          <th>Quantity</th>
          <th>Value (USD)</th>
        </tr>
      </thead>
      <tbody></tbody>
    </table>

    <h2>Price Alerts</h2>
    <ul id="alertsList"></ul>
  </div>

  <script>
    const apiUrl = 'https://api.coingecko.com/api/v3/coins/markets?vs_currency=usd&per_page=50&page=1';
    let portfolio = JSON.parse(localStorage.getItem("portfolio")) || {};
    let alerts = JSON.parse(localStorage.getItem("alerts")) || [];
    let coinData = [];
    let sortState = { key: 'name', asc: true };

    function updateLocalStorage() {
      localStorage.setItem("portfolio", JSON.stringify(portfolio));
      localStorage.setItem("alerts", JSON.stringify(alerts));
    }

    function fetchCoins() {
      fetch(apiUrl)
        .then(res => res.json())
        .then(data => {
          coinData = data;
          renderTable();
          updatePortfolioDisplay();
          checkAlerts();
        });
    }

    function renderTable() {
      const tableBody = document.querySelector("#coinTable tbody");
      tableBody.innerHTML = '';
      const search = document.getElementById("searchInput").value.toLowerCase();
      let filtered = coinData.filter(c => c.name.toLowerCase().includes(search));

      filtered.sort((a, b) => {
        let key = sortState.key === 'name' ? 'name' : 'current_price';
        let dir = sortState.asc ? 1 : -1;
        return (a[key] > b[key] ? 1 : -1) * dir;
      });

      filtered.forEach(coin => {
        const row = document.createElement("tr");
        row.innerHTML = `
          <td>${coin.name}</td>
          <td>$${coin.current_price.toLocaleString()}</td>
          <td><input type="number" id="qty-${coin.id}" placeholder="Qty"></td>
          <td><button onclick="addToPortfolio('${coin.id}', '${coin.name}', ${coin.current_price})">Add</button></td>
          <td>
            <input type="number" id="alert-${coin.id}" placeholder="Target Price">
            <button onclick="setAlert('${coin.id}', '${coin.name}', ${coin.current_price})">Set</button>
          </td>`;
        tableBody.appendChild(row);
      });
    }

    function sortTable(key) {
      sortState.asc = sortState.key === key ? !sortState.asc : true;
      sortState.key = key;
      renderTable();
    }

    function addToPortfolio(id, name, price) {
      const qty = parseFloat(document.getElementById(`qty-${id}`).value);
      if (!qty || qty <= 0) return alert("Invalid quantity");
      if (!portfolio[id]) portfolio[id] = { name, qty: 0 };
      portfolio[id].qty += qty;
      updateLocalStorage();
      updatePortfolioDisplay();
    }

    function updatePortfolioDisplay() {
      const tableBody = document.querySelector("#portfolioTable tbody");
      tableBody.innerHTML = '';
      let total = 0;
      for (let id in portfolio) {
        const coin = coinData.find(c => c.id === id);
        if (!coin) continue;
        const value = coin.current_price * portfolio[id].qty;
        total += value;
        const row = document.createElement("tr");
        row.innerHTML = `<td>${portfolio[id].name}</td><td>${portfolio[id].qty}</td><td>$${value.toFixed(2)}</td>`;
        tableBody.appendChild(row);
      }
      const row = document.createElement("tr");
      row.innerHTML = `<td colspan="2"><strong>Total</strong></td><td><strong>$${total.toFixed(2)}</strong></td>`;
      tableBody.appendChild(row);
    }

    function setAlert(id, name, price) {
      const target = parseFloat(document.getElementById(`alert-${id}`).value);
      if (!target || target <= 0) return alert("Invalid target");
      alerts.push({ id, name, target, triggered: false });
      updateLocalStorage();
      displayAlerts();
    }

    function displayAlerts() {
      const list = document.getElementById("alertsList");
      list.innerHTML = '';
      alerts.forEach((a, i) => {
        const li = document.createElement("li");
        li.classList.add("alert-item");
        li.innerHTML = `${a.name} → Target: $${a.target.toLocaleString()} <button onclick="removeAlert(${i})">Remove</button>`;
        list.appendChild(li);
      });
    }

    function removeAlert(index) {
      alerts.splice(index, 1);
      updateLocalStorage();
      displayAlerts();
    }

    function checkAlerts() {
      alerts.forEach(a => {
        const coin = coinData.find(c => c.id === a.id);
        if (coin && !a.triggered && coin.current_price >= a.target) {
          a.triggered = true;
          notify(`${a.name} hit $${a.target.toLocaleString()}!`);
        }
      });
      updateLocalStorage();
    }

    function notify(message) {
      if ("Notification" in window) {
        Notification.requestPermission().then(p => {
          if (p === "granted") {
            new Notification("Crypto Alert", { body: message });
          } else alert(message);
        });
      } else alert(message);
    }

    fetchCoins();
    displayAlerts();
    setInterval(fetchCoins, 30000);
  </script>
</body>
</html>
