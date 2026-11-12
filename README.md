import sqlite3
import ipywidgets as widgets
from IPython.display import display, HTML, clear_output
from datetime import date
from html import escape

# ================= DATABASE =================

conn = sqlite3.connect("industrial_waste_final.db", check_same_thread=False)

conn.execute("""
CREATE TABLE IF NOT EXISTS waste (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    waste_name TEXT,
    industry TEXT,
    waste_type TEXT,
    quantity REAL,
    disposal TEXT,
    location TEXT,
    date TEXT
)
""")
conn.commit()

# ================= CSS DESIGN =================

css = """
<style>
.main {
    font-family: Arial;
    background: #f4f7f6;
    padding: 20px;
    border-radius: 15px;
}
.header {
    background: linear-gradient(120deg,#087443,#20a36b);
    color: white;
    text-align: center;
    padding: 25px;
    border-radius: 12px;
}
.cards {
    display: flex;
    gap: 15px;
    flex-wrap: wrap;
    margin: 20px 0;
}
.card {
    flex: 1;
    min-width: 150px;
    background: white;
    padding: 20px;
    border-radius: 12px;
    text-align: center;
    box-shadow: 0 2px 8px #cccccc;
}
.card h2 { color: #087443; font-size: 28px; }
.card p { color: #555; font-weight: bold; }
.section {
    background: white;
    padding: 18px;
    border-radius: 12px;
    margin-top: 18px;
}
.main table {
    width: 100%;
    border-collapse: collapse;
    background: white;
    font-size: 13px;
}
.main th {
    background: #087443;
    color: white;
    padding: 12px 7px;
}
.main td {
    border: 1px solid #ddd;
    padding: 10px 6px;
    text-align: center;
}
.main tr:nth-child(even) { background: #eef8f2; }
.status {
    padding: 12px;
    background: #d5f5e3;
    color: #087443;
    border-radius: 8px;
    font-weight: bold;
}
</style>
"""

# ================= INPUT FIELDS =================

waste_name = widgets.Text(
    description="Waste Name:",
    placeholder="Enter waste name",
    layout=widgets.Layout(width="95%")
)

industry = widgets.Text(
    description="Industry:",
    placeholder="Enter industry name",
    layout=widgets.Layout(width="95%")
)

waste_type = widgets.Dropdown(
    options=["Solid Waste", "Liquid Waste",
             "Plastic Waste", "Organic Waste",
             "Hazardous Waste"],
    description="Waste Type:",
    layout=widgets.Layout(width="95%")
)

quantity = widgets.FloatText(
    description="Quantity:",
    value=100,
    layout=widgets.Layout(width="95%")
)

disposal = widgets.Dropdown(
    options=["Recycling", "Landfill", "Incineration",
             "Composting", "Chemical Treatment",
             "Authorized Treatment Facility"],
    description="Disposal:",
    layout=widgets.Layout(width="95%")
)

location = widgets.Text(
    description="Location:",
    placeholder="Enter location",
    layout=widgets.Layout(width="95%")
)

waste_date = widgets.DatePicker(
    description="Date:",
    value=date.today(),
    layout=widgets.Layout(width="95%")
)

search_box = widgets.Text(
    placeholder="Search waste name or industry...",
    layout=widgets.Layout(width="75%")
)

record_id = widgets.IntText(
    description="Record ID:",
    value=1,
    layout=widgets.Layout(width="95%")
)

new_quantity = widgets.FloatText(
    description="New Qty:",
    value=200,
    layout=widgets.Layout(width="95%")
)

# ================= BUTTONS =================

add_btn = widgets.Button(
    description="ADD WASTE",
    button_style="success",
    icon="plus",
    layout=widgets.Layout(width="180px")
)

search_btn = widgets.Button(
    description="SEARCH",
    button_style="info",
    icon="search"
)

show_btn = widgets.Button(
    description="SHOW ALL",
    button_style="primary"
)

delete_btn = widgets.Button(
    description="DELETE",
    button_style="danger"
)

update_btn = widgets.Button(
    description="UPDATE QUANTITY",
    button_style="warning"
)

output = widgets.Output()

# ================= DASHBOARD =================

def dashboard(keyword="", message=""):

    total = conn.execute(
        "SELECT COUNT(*) FROM waste"
    ).fetchone()[0]

    qty = conn.execute(
        "SELECT SUM(quantity) FROM waste"
    ).fetchone()[0] or 0

    categories = conn.execute(
        "SELECT COUNT(DISTINCT waste_type) FROM waste"
    ).fetchone()[0]

    if keyword:
        records = conn.execute("""
            SELECT * FROM waste
            WHERE waste_name LIKE ? OR industry LIKE ?
            ORDER BY id DESC
        """, ('%' + keyword + '%', '%' + keyword + '%')).fetchall()
    else:
        records = conn.execute(
            "SELECT * FROM waste ORDER BY id DESC"
        ).fetchall()

    table_rows = ""

    for r in records:
        table_rows += "<tr>" + "".join(
            f"<td>{escape(str(x))}</td>" for x in r
        ) + "</tr>"

    if not table_rows:
        table_rows = """
        <tr><td colspan="8">No records found. Please add waste details.</td></tr>
        """

    html = css + f"""
    <div class="main">

    <div class="header">
      <h1>INDUSTRIAL WASTE MANAGEMENT</h1>
      <p>Processing and Disposal Management System</p>
      <p>Smart Waste Monitoring Dashboard</p>
    </div>

    <div class="cards">
      <div class="card">
        <p>TOTAL RECORDS</p>
        <h2>{total}</h2>
      </div>

      <div class="card">
        <p>TOTAL WASTE</p>
        <h2>{qty:,.2f} kg</h2>
      </div>

      <div class="card">
        <p>WASTE CATEGORIES</p>
        <h2>{categories}</h2>
      </div>
    </div>

    <div class="section">
      <h2 style="color:#087443">Waste Records</h2>
      <p>All industrial waste details stored in database</p>

      <table>
        <tr>
          <th>ID</th>
          <th>Waste Name</th>
          <th>Industry</th>
          <th>Type</th>
          <th>Quantity</th>
          <th>Disposal</th>
          <th>Location</th>
          <th>Date</th>
        </tr>
        {table_rows}
      </table>
    </div>

    {f'<p class="status">{escape(message)}</p>' if message else ''}

    </div>
    """

    with output:
        clear_output(wait=True)
        display(HTML(html))


# ================= ADD RECORD =================

def add_record(b):

    if not waste_name.value.strip() or not industry.value.strip() or not location.value.strip():
        dashboard(message="Please fill all required fields!")
        return

    if quantity.value <= 0:
        dashboard(message="Quantity must be greater than zero!")
        return

    if waste_date.value is None:
        dashboard(message="Please select a date!")
        return

    conn.execute("""
    INSERT INTO waste
    (waste_name, industry, waste_type, quantity,
     disposal, location, date)
    VALUES (?, ?, ?, ?, ?, ?, ?)
    """, (
        waste_name.value.strip(),
        industry.value.strip(),
        waste_type.value,
        quantity.value,
        disposal.value,
        location.value.strip(),
        waste_date.value.isoformat()
    ))

    conn.commit()

    dashboard(message="Waste record added successfully!")

    waste_name.value = ""
    industry.value = ""
    quantity.value = 100
    location.value = ""


# ================= SEARCH =================

def search_record(b):
    dashboard(keyword=search_box.value.strip(),
              message="Search completed!")


# ================= SHOW ALL =================

def show_all(b):
    search_box.value = ""
    dashboard(message="All records displayed!")


# ================= DELETE =================

def delete_record(b):

    cur = conn.execute(
        "DELETE FROM waste WHERE id=?",
        (record_id.value,)
    )

    conn.commit()

    if cur.rowcount:
        dashboard(message="Record deleted successfully!")
    else:
        dashboard(message="Record ID not found!")


# ================= UPDATE =================

def update_record(b):

    if new_quantity.value <= 0:
        dashboard(message="Enter a valid quantity!")
        return

    cur = conn.execute(
        "UPDATE waste SET quantity=? WHERE id=?",
        (new_quantity.value, record_id.value)
    )

    conn.commit()

    if cur.rowcount:
        dashboard(message="Quantity updated successfully!")
    else:
        dashboard(message="Record ID not found!")


# ================= BUTTON ACTIONS =================

add_btn.on_click(add_record)
search_btn.on_click(search_record)
show_btn.on_click(show_all)
delete_btn.on_click(delete_record)
update_btn.on_click(update_record)

# ================= FINAL INTERFACE =================

form = widgets.VBox([
    widgets.HTML("<h2 style='color:#087443'>ADD INDUSTRIAL WASTE DETAILS</h2>"),
    waste_name,
    industry,
    waste_type,
    quantity,
    disposal,
    location,
    waste_date,
    add_btn,

    widgets.HTML("<hr><h2 style='color:#087443'>SEARCH RECORDS</h2>"),
    search_box,
    widgets.HBox([search_btn, show_btn]),

    widgets.HTML("<hr><h2 style='color:#087443'>UPDATE / DELETE RECORD</h2>"),
    record_id,
    new_quantity,
    widgets.HBox([update_btn, delete_btn])
])

display(form)
display(output)

dashboard(message="Welcome! Add your first industrial waste record.")
