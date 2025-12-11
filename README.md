import React, { useEffect, useMemo, useState } from "react";

// Personal Finance Manager - Single-file React component
// Requirements: TailwindCSS for styling. Put this component in a React app (Create React App / Vite) and ensure Tailwind is configured.

export default function PersonalFinanceApp() {
  // Data model
  // transactions: { id, type: 'income'|'expense', date: 'YYYY-MM-DD', amount: number, description, category }

  const LS_TX = "pfm_transactions_v1";
  const LS_SB = "pfm_starting_balance_v1";

  const [startingBalance, setStartingBalance] = useState(0);
  const [transactions, setTransactions] = useState([]);
  const [viewMonth, setViewMonth] = useState(() => {
    const d = new Date();
    return `${d.getFullYear()}-${String(d.getMonth() + 1).padStart(2, "0")}`; // YYYY-MM
  });

  // Form states
  const [isAdding, setIsAdding] = useState(false);
  const [form, setForm] = useState({ type: "expense", date: todayISO(), amount: "", description: "", category: "" });

  // Load from localStorage
  useEffect(() => {
    try {
      const rawTx = localStorage.getItem(LS_TX);
      const rawSb = localStorage.getItem(LS_SB);
      if (rawTx) setTransactions(JSON.parse(rawTx));
      if (rawSb) setStartingBalance(Number(rawSb));
    } catch (e) {
      console.warn("Failed to load from localStorage", e);
    }
  }, []);

  useEffect(() => {
    localStorage.setItem(LS_TX, JSON.stringify(transactions));
  }, [transactions]);

  useEffect(() => {
    localStorage.setItem(LS_SB, String(startingBalance));
  }, [startingBalance]);

  // Derived values
  const totals = useMemo(() => {
    let totalIncome = 0;
    let totalExpense = 0;
    for (const t of transactions) {
      if (t.type === "income") totalIncome += Number(t.amount);
      else totalExpense += Number(t.amount);
    }
    const currentBalance = Number(startingBalance) + totalIncome - totalExpense;
    return { totalIncome, totalExpense, currentBalance };
  }, [transactions, startingBalance]);

  // Monthly view derived
  const monthlyData = useMemo(() => {
    const [y, m] = viewMonth.split("-").map(Number);
    const monthStart = new Date(y, m - 1, 1);
    const monthEnd = new Date(y, m, 0, 23, 59, 59);
    const tx = transactions.filter((t) => {
      const d = new Date(t.date + "T00:00:00");
      return d >= monthStart && d <= monthEnd;
    });
    let added = 0;
    let spent = 0;
    for (const t of tx) {
      if (t.type === "income") added += Number(t.amount);
      else spent += Number(t.amount);
    }
    // Sort by date desc
    tx.sort((a, b) => (a.date < b.date ? 1 : -1));
    return { tx, added, spent };
  }, [transactions, viewMonth]);

  // Helpers
  function todayISO() {
    const d = new Date();
    return `${d.getFullYear()}-${String(d.getMonth() + 1).padStart(2, "0")}-${String(d.getDate()).padStart(2, "0")}`;
  }

  function formatCurrency(n) {
    const num = Number(n) || 0;
    return num.toLocaleString(undefined, { style: "currency", currency: "USD", minimumFractionDigits: 2 });
  }

  // Actions
  function addTransaction(e) {
    e?.preventDefault();
    if (!form.date || !form.amount) return;
    const entry = {
      id: cryptoRandomId(),
      type: form.type,
      date: form.date,
      amount: Number(form.amount),
      description: form.description || (form.type === "income" ? "Income" : "Expense"),
      category: form.category || "Uncategorized",
    };
    setTransactions((p) => [entry, ...p]);
    // Quick UX: if income, immediately affect current balance via derived state
    setForm({ type: "expense", date: todayISO(), amount: "", description: "", category: "" });
    setIsAdding(false);
  }

  function deleteTransaction(id) {
    if (!confirm("Delete this transaction?")) return;
    setTransactions((p) => p.filter((t) => t.id !== id));
  }

  function setInitialStartingBalance(value) {
    const n = Number(value) || 0;
    setStartingBalance(n);
  }

  // small unique id
  function cryptoRandomId() {
    return Math.random().toString(36).slice(2, 9);
  }

  // Month navigation helpers
  function changeMonth(offset) {
    const [y, m] = viewMonth.split("-").map(Number);
    const d = new Date(y, m - 1 + offset, 1);
    setViewMonth(`${d.getFullYear()}-${String(d.getMonth() + 1).padStart(2, "0")}`);
  }

  return (
    <div className="min-h-screen bg-slate-50 p-4 md:p-8">
      <div className="max-w-4xl mx-auto">
        <header className="flex items-center justify-between mb-6">
          <h1 className="text-2xl md:text-3xl font-semibold">Personal Finance — Monthly Overview</h1>
          <div className="text-right text-sm text-slate-600">Mobile-responsive • Quick add • Local-only data</div>
        </header>

        {/* Balance and quick controls */}
        <section className="grid grid-cols-1 md:grid-cols-3 gap-4 mb-6">
          <div className="md:col-span-2 bg-white rounded-2xl p-4 shadow-sm">
            <div className="flex items-center justify-between">
              <div>
                <div className="text-sm text-slate-500">Current Balance</div>
                <div className="mt-1 text-2xl font-bold">{formatCurrency(totals.currentBalance)}</div>
                <div className="mt-2 text-xs text-slate-500">Starting Balance: {formatCurrency(startingBalance)}</div>
              </div>
              <div className="flex items-center gap-2">
                <button
                  className="px-3 py-2 rounded-lg border hover:bg-slate-50 text-sm"
                  onClick={() => {
                    const input = prompt("Set starting balance (numbers only)", String(startingBalance));
                    if (input !== null) setInitialStartingBalance(input);
                  }}
                >
                  Set Starting Balance
                </button>
                <button
                  className="px-3 py-2 rounded-lg bg-green-600 text-white text-sm shadow"
                  onClick={() => {
                    setForm({ ...form, type: "income", date: todayISO() });
                    setIsAdding(true);
                  }}
                >
                  + Add Income
                </button>
                <button
                  className="px-3 py-2 rounded-lg bg-red-600 text-white text-sm shadow"
                  onClick={() => {
                    setForm({ ...form, type: "expense", date: todayISO() });
                    setIsAdding(true);
                  }}
                >
                  - Add Expense
                </button>
              </div>
            </div>
          </div>

          <div className="bg-white rounded-2xl p-4 shadow-sm flex flex-col gap-3">
            <div className="flex items-center justify-between">
              <div className="text-sm text-slate-500">Selected Month</div>
              <div className="text-sm text-slate-500">Navigation</div>
            </div>
            <div className="flex items-center gap-2">
              <button className="px-2 py-1 rounded border" onClick={() => changeMonth(-1)}>Prev</button>
              <input
                type="month"
                value={viewMonth}
                onChange={(e) => setViewMonth(e.target.value)}
                className="rounded px-2 py-1 border text-sm"
                aria-label="Choose month"
              />
              <button className="px-2 py-1 rounded border" onClick={() => changeMonth(1)}>Next</button>
            </div>
            <div className="pt-2 border-t mt-2 text-sm">
              <div className="flex justify-between">
                <div>Added</div>
                <div className="font-medium text-green-600">{formatCurrency(monthlyData.added)}</div>
              </div>
              <div className="flex justify-between mt-1">
                <div>Spent</div>
                <div className="font-medium text-red-600">{formatCurrency(monthlyData.spent)}</div>
              </div>
            </div>
          </div>
        </section>

        {/* Add form modal (small) */}
        {isAdding && (
          <div className="fixed inset-0 bg-black/30 flex items-center justify-center p-4">
            <form onSubmit={addTransaction} className="bg-white rounded-2xl p-4 w-full max-w-md shadow">
              <div className="flex items-center justify-between mb-3">
                <h3 className="text-lg font-semibold">{form.type === "income" ? "Add Income" : "Add Expense"}</h3>
                <button type="button" className="text-slate-500" onClick={() => setIsAdding(false)}>Close</button>
              </div>
              <div className="grid gap-2">
                <label className="text-xs text-slate-600">Date</label>
                <input type="date" value={form.date} onChange={(e) => setForm({ ...form, date: e.target.value })} className="px-3 py-2 border rounded" required />

                <label className="text-xs text-slate-600">Amount</label>
                <input type="number" step="0.01" value={form.amount} onChange={(e) => setForm({ ...form, amount: e.target.value })} className="px-3 py-2 border rounded" required />

                <label className="text-xs text-slate-600">Description / Item</label>
                <input type="text" value={form.description} onChange={(e) => setForm({ ...form, description: e.target.value })} placeholder="e.g., Salary, Groceries" className="px-3 py-2 border rounded" />

                <label className="text-xs text-slate-600">Category (optional)</label>
                <input type="text" value={form.category} onChange={(e) => setForm({ ...form, category: e.target.value })} placeholder="e.g., Food, Bills" className="px-3 py-2 border rounded" />

                <div className="flex gap-2 mt-2">
                  <button type="submit" className="flex-1 py-2 rounded bg-blue-600 text-white">Save</button>
                  <button type="button" className="flex-1 py-2 rounded border" onClick={() => setIsAdding(false)}>Cancel</button>
                </div>
              </div>
            </form>
          </div>
        )}

        {/* Transactions list */}
        <section className="bg-white rounded-2xl p-4 shadow-sm mt-6">
          <div className="flex items-center justify-between mb-3">
            <h2 className="text-lg font-semibold">Transactions — {viewMonth}</h2>
            <div className="text-sm text-slate-500">{monthlyData.tx.length} items</div>
          </div>
          <div className="divide-y">
            {monthlyData.tx.length === 0 && (
              <div className="text-sm text-slate-500 py-6 text-center">No transactions for this month. Use the buttons above to add income or expenses quickly.</div>
            )}
            {monthlyData.tx.map((t) => (
              <div key={t.id} className="flex items-center justify-between py-3">
                <div className="flex items-center gap-3">
                  <div className={`w-10 h-10 rounded-xl flex items-center justify-center text-white font-semibold ${t.type === "income" ? "bg-green-600" : "bg-red-600"}`}>
                    {t.type === "income" ? "+" : "-"}
                  </div>
                  <div>
                    <div className="text-sm font-medium">{t.description}</div>
                    <div className="text-xs text-slate-500">{t.category} • {t.date}</div>
                  </div>
                </div>
                <div className="flex items-center gap-3">
                  <div className={`text-sm font-semibold ${t.type === "income" ? "text-green-600" : "text-red-600"}`}>{t.type === "income" ? "+" : "-"}{formatCurrency(t.amount)}</div>
                  <button className="text-xs text-slate-400 hover:text-slate-700" onClick={() => deleteTransaction(t.id)}>Delete</button>
                </div>
              </div>
            ))}
          </div>
        </section>

        {/* Small footer / summary */}
        <footer className="text-xs text-slate-500 text-center mt-6">Data stored locally in your browser. Exporting, cloud sync, authentication can be added later.</footer>
      </div>
    </div>
  );
}

