<!DOCTYPE html>  
<html lang="en">  
<head>  
  <meta charset="UTF-8" />  
  <meta name="viewport" content="width=device-width, initial-scale=1.0" />  
  <title>Zstores - Hospital Order Dispatch</title>  
  
  <!-- React & ReactDOM CDN -->  
  <script src="https://unpkg.com/react@18/umd/react.production.min.js" crossorigin></script>  
  <script src="https://unpkg.com/react-dom@18/umd/react-dom.production.min.js" crossorigin></script>  
    
  <!-- Babel CDN for inline JSX -->  
  <script src="https://unpkg.com/@babel/standalone/babel.min.js"></script>  
    
  <!-- PDF.js CDN -->  
  <script src="https://cdnjs.cloudflare.com/ajax/libs/pdf.js/3.11.174/pdf.min.js"></script>  
  
  <!-- Tailwind CSS CDN -->  
  <script src="https://cdn.tailwindcss.com"></script>  
  
  <style>  
    @media print {  
      .no-print { display: none !important; }  
      .print-only { display: block !important; }  
      body { background: white; color: black; font-size: 12pt; }  
      .page-break { page-break-after: always; }  
      .card { border: 1px solid #ccc !important; shadow: none !important; margin-bottom: 1rem; }  
    }  
  </style>  
</head>  
<body class="bg-slate-100 min-h-screen text-slate-800">  
  <div id="root"></div>  
  
  <script type="text/babel">  
    const { useState, useEffect, useMemo } = React;  
  
    // Configure PDF.js worker  
    pdfjsLib.GlobalWorkerOptions.workerSrc = 'https://cdnjs.cloudflare.com/ajax/libs/pdf.js/3.11.174/pdf.worker.min.js';  
  
    // Default Categorization Rules (Exact match keywords)  
    const DEFAULT_RULES = {  
      "Juice Run": [  
        "Apple Juice PC Ripe Road", "Orange Juice PC - Ripe Road",   
        "Diced Peaches PC - Riviana", "Diced Pears PC - Riviana"  
      ],  
      "Chiller Run": [  
        "Full Cream Milk - Brownes (2 Litre)", "Full Cream Milk PC - Brownes",   
        "Hilo Milk PC- Brownes", "Hilo Milk - Brownes (2 Litre)",   
        "Vanilla Yoghurt PC - Brownes", "Strawberry Yoghurt PC Brownes",   
        "Rice Pudding - Birch & Waite", "Custard PC - Birch & Waite",   
        "Cheese - Cheddar PC - Vepo"  
      ],  
      "Dry Run": [  
        "White Sugar PC - Bundaberg", "Coffee PC - Nescafe Blend 43",   
        "Decaf Coffee PC - Nescafe", "Sweetener Sachet PC - Equal",   
        "Water Crackers PC - Arnotts", "Scotch Nice Biscuit PC - Arnotts",   
        "Soy Milk - Sanitarium So Good", "Rice Milk - Vitasoy",   
        "Lactose Free Milk - Low fat - Little Things", "Tea Bags Black - Lipton",   
        "Green Tea Bags - Lipton", "Tea Bags - Bushells",   
        "Fruit Cake PC - Menumaster", "Milo - Nestle"  
      ],  
      "Freezer Run": [  
        "Wholemeal Bread - SW- Le Moulin", "White Bread - Le Moulin",   
        "Multigrain Bread - SW- Le Moulin", "Ice Cream PC - Bulla",   
        "Butter PC - Devondale", "Margarine PC - Meadowlea Milk Free"  
      ],  
      "Afternoon Run": [  
        "Apple Fresh", "Bananas fresh", "Carrot Cake - Homebush Cakes"  
      ]  
    };  
  
    function App() {  
      const [rules, setRules] = useState(() => {  
        const saved = localStorage.getItem('zstores_rules');  
        return saved ? JSON.parse(saved) : DEFAULT_RULES;  
      });  
  
      const [parsedData, setParsedData] = useState([]);  
      const [selectedRun, setSelectedRun] = useState("Juice Run");  
      const [checkedItems, setCheckedItems] = useState({});  
      const [activeTab, setActiveTab] = useState("picker"); // picker | settings  
      const [newCategoryItem, setNewCategoryItem] = useState("");  
      const [targetCategory, setTargetCategory] = useState("Juice Run");  
      const [isLoading, setIsLoading] = useState(false);  
  
      useEffect(() => {  
        localStorage.setItem('zstores_rules', JSON.stringify(rules));  
      }, [rules]);  
  
      // Categorize item based on current rules  
      const getCategory = (itemName) => {  
        for (const [category, keywords] of Object.entries(rules)) {  
          if (keywords.some(kw => itemName.toLowerCase().includes(kw.toLowerCase()))) {  
            return category;  
          }  
        }  
        return "Pantry Run"; // Fallback for unmatched items  
      };  
  
      // Handle PDF Upload and Extraction  
      const handleFileUpload = async (event) => {  
        const file = event.target.files[0];  
        if (!file) return;  
  
        setIsLoading(true);  
        try {  
          const arrayBuffer = await file.arrayBuffer();  
          const pdf = await pdfjsLib.getDocument({ data: arrayBuffer }).promise;  
          let fullTextLines = [];  
  
          for (let i = 1; i <= pdf.numPages; i++) {  
            const page = await pdf.getPage(i);  
            const textContent = await page.getTextContent();  
            const pageLines = textContent.items.map(item => item.str.trim()).filter(str => str !== "");  
            fullTextLines = [...fullTextLines, ...pageLines];  
          }  
  
          const parsedOrders = parseDeliveryNoteLines(fullTextLines);  
          setParsedData(parsedOrders);  
          setCheckedItems({});  
        } catch (err) {  
          alert("Error parsing PDF file: " + err.message);  
        } finally {  
          setIsLoading(false);  
        }  
      };  
  
      // Custom Regex Parser for Delivery Note Layouts  
      const parseDeliveryNoteLines = (lines) => {  
        const orders = [];  
        let currentWard = "Unknown Ward";  
          
        for (let i = 0; i < lines.length; i++) {  
          const line = lines[i];  
  
          // Detect Ward Header (e.g. 2/09/2026 / FSH-3AB Ad Hoc / ATG2609-04000)  
          if (line.includes("FSH-")) {  
            const match = line.match(/(FSH-[A-Za-z0-9\s\/]+?)(?=\s+Ad|\s+\/|$)/);  
            if (match) {  
              currentWard = match[1].trim();  
            } else {  
              currentWard = line.trim();  
            }  
            continue;  
          }  
  
          // Detect Items: Looking for quantity pattern followed by unit and component name  
          // E.g., POS (1), Qty (2), Unit (Each), Component Name (Spoons Wooden...)  
          if (line.match(/^\d+$/) && i + 3 < lines.length) {  
            const possibleQty = lines[i + 1];  
            const possibleUnit = lines[i + 2];  
            const possibleName = lines[i + 3];  
  
            // Verify if line + 1 is numeric quantity  
            if (possibleQty.match(/^\d+$/) && (possibleUnit === "Each" || possibleUnit === "Litre" || possibleUnit === "Kilogram")) {  
              orders.push({  
                id: `${currentWard}-${possibleName}-${Math.random()}`,  
                ward: currentWard,  
                qty: parseInt(possibleQty, 10),  
                unit: possibleUnit,  
                name: possibleName,  
                category: getCategory(possibleName)  
              });  
              i += 3; // Skip processed elements  
            }  
          }  
        }  
  
        return orders;  
      };  
  
      // Toggle item checklist state  
      const toggleCheck = (id) => {  
        setCheckedItems(prev => ({ ...prev, [id]: !prev[id] }));  
      };  
  
      // Filter items for current selected category  
      const currentRunItems = useMemo(() => {  
        return parsedData.filter(item => item.category === selectedRun);  
      }, [parsedData, selectedRun]);  
  
      // Group items by Ward  
      const itemsByWard = useMemo(() => {  
        const grouped = {};  
        currentRunItems.forEach(item => {  
          if (!grouped[item.ward]) grouped[item.ward] = [];  
          grouped[item.ward].push(item);  
        });  
        return grouped;  
      }, [currentRunItems]);  
  
      // Add new item rule  
      const handleAddItemRule = (e) => {  
        e.preventDefault();  
        if (!newCategoryItem.trim()) return;  
  
        setRules(prev => ({  
          ...prev,  
          [targetCategory]: [...(prev[targetCategory] || []), newCategoryItem.trim()]  
        }));  
        setNewCategoryItem("");  
      };  
  
      // Remove rule  
      const handleRemoveRule = (category, keyword) => {  
        setRules(prev => ({  
          ...prev,  
          [category]: prev[category].filter(k => k !== keyword)  
        }));  
      };  
  
      return (  
        <div className="max-w-6xl mx-auto p-4 md:p-6">  
          {/* App Header */}  
          <header className="no-print bg-slate-900 text-white p-4 rounded-xl shadow-lg mb-6 flex flex-col md:flex-row justify-between items-center gap-4">  
            <div>  
              <h1 className="text-2xl font-bold tracking-wide text-emerald-400">Zstores</h1>  
              <p className="text-slate-400 text-xs">Hospital Store Category Run Dispatch System</p>  
            </div>  
  
            <div className="flex items-center gap-3">  
              <button   
                onClick={() => setActiveTab("picker")}  
                className={`px-4 py-2 text-sm font-semibold rounded-lg transition ${activeTab === 'picker' ? 'bg-emerald-500 text-slate-950' : 'bg-slate-800 text-slate-300 hover:bg-slate-700'}`}  
              >  
                Run Picker  
              </button>  
              <button   
                onClick={() => setActiveTab("settings")}  
                className={`px-4 py-2 text-sm font-semibold rounded-lg transition ${activeTab === 'settings' ? 'bg-emerald-500 text-slate-950' : 'bg-slate-800 text-slate-300 hover:bg-slate-700'}`}  
              >  
                Item Rules Settings  
              </button>  
            </div>  
          </header>  
  
          {activeTab === "picker" ? (  
            <div>  
              {/* PDF Upload & Controls Bar */}  
              <div className="no-print bg-white p-4 rounded-xl shadow-sm mb-6 flex flex-col md:flex-row justify-between items-center gap-4">  
                <div className="w-full md:w-auto">  
                  <label className="block text-xs font-semibold uppercase text-slate-500 mb-1">Upload Daily Delivery Note PDF</label>  
                  <input   
                    type="file"   
                    accept="application/pdf"  
                    onChange={handleFileUpload}  
                    className="block w-full text-sm text-slate-500 file:mr-4 file:py-2 file:px-4 file:rounded-lg file:border-0 file:text-sm file:font-semibold file:bg-emerald-50 file:text-emerald-700 hover:file:bg-emerald-100 cursor-pointer"  
                  />  
                </div>  
  
                {parsedData.length > 0 && (  
                  <button   
                    onClick={() => window.print()}   
                    className="w-full md:w-auto px-5 py-2.5 bg-slate-800 hover:bg-slate-700 text-white text-sm font-medium rounded-lg flex items-center justify-center gap-2 shadow"  
                  >  
                    <span>🖨️ Print Active Run List</span>  
                  </button>  
                )}  
              </div>  
  
              {isLoading && (  
                <div className="bg-white p-8 rounded-xl shadow-sm text-center font-medium text-slate-600">  
                  Processing PDF Delivery Note...  
                </div>  
              )}  
  
              {/* Category Run Selector Tabs */}  
              {parsedData.length > 0 && (  
                <div className="no-print grid grid-cols-2 md:grid-cols-6 gap-2 mb-6">  
                  {["Juice Run", "Chiller Run", "Dry Run", "Freezer Run", "Afternoon Run", "Pantry Run"].map(category => {  
                    const count = parsedData.filter(i => i.category === category).length;  
                    const isSelected = selectedRun === category;  
                    return (  
                      <button  
                        key={category}  
                        onClick={() => setSelectedRun(category)}  
                        className={`p-3 rounded-xl border text-left transition relative ${  
                          isSelected   
                            ? 'bg-emerald-600 text-white border-emerald-600 shadow-md'   
                            : 'bg-white text-slate-700 border-slate-200 hover:border-slate-300'  
                        }`}  
                      >  
                        <div className="text-xs font-medium opacity-80">{category}</div>  
                        <div className="text-xl font-bold mt-1">{count} <span className="text-xs font-normal">items</span></div>  
                      </button>  
                    );  
                  })}  
                </div>  
              )}  
  
              {/* Active Run Delivery Checklist Display */}  
              {parsedData.length > 0 ? (  
                <div>  
                  <div className="mb-4 flex justify-between items-end">  
                    <div>  
                      <h2 className="text-xl font-bold text-slate-800">{selectedRun} Delivery List</h2>  
                      <p className="text-xs text-slate-500">Showing items sorted by ward locations</p>  
                    </div>  
                    <span className="text-xs font-medium bg-slate-200 text-slate-700 px-3 py-1 rounded-full">  
                      {currentRunItems.length} Total Units  
                    </span>  
                  </div>  
  
                  {Object.keys(itemsByWard).length === 0 ? (  
                    <div className="bg-white p-8 rounded-xl shadow-sm text-center text-slate-500">  
                      No items requested for {selectedRun} in today's delivery note.  
                    </div>  
                  ) : (  
                    <div className="grid gap-4">  
                      {Object.entries(itemsByWard).map(([ward, items]) => (  
                        <div key={ward} className="card bg-white rounded-xl shadow-sm p-4 border border-slate-200">  
                          <h3 className="text-lg font-bold text-slate-900 border-b pb-2 mb-3 flex justify-between items-center">  
                            <span>{ward}</span>  
                            <span className="text-xs bg-emerald-100 text-emerald-800 font-semibold px-2.5 py-0.5 rounded-full">  
                              {items.length} items  
                            </span>  
                          </h3>  
  
                          <div className="divide-y divide-slate-100">  
                            {items.map(item => {  
                              const isChecked = !!checkedItems[item.id];  
                              return (  
                                <div   
                                  key={item.id}   
                                  onClick={() => toggleCheck(item.id)}  
                                  className={`py-2.5 px-2 flex items-center justify-between cursor-pointer rounded transition ${isChecked ? 'bg-slate-50 opacity-60' : 'hover:bg-slate-50'}`}  
                                >  
                                  <div className="flex items-center gap-3">  
                                    <input   
                                      type="checkbox"   
                                      checked={isChecked}  
                                      onChange={() => {}}   
                                      className="w-5 h-5 accent-emerald-600 rounded cursor-pointer"  
                                    />  
                                    <span className={`text-sm font-medium ${isChecked ? 'line-through text-slate-400' : 'text-slate-800'}`}>  
                                      {item.name}  
                                    </span>  
                                  </div>  
  
                                  <div className="text-sm font-bold text-slate-900 bg-slate-100 px-3 py-1 rounded-md">  
                                    {item.qty} <span className="text-xs font-normal text-slate-500">{item.unit}</span>  
                                  </div>  
                                </div>  
                              );  
                            })}  
                          </div>  
                        </div>  
                      ))}  
                    </div>  
                  )}  
                </div>  
              ) : (  
                !isLoading && (  
                  <div className="bg-white p-12 rounded-xl shadow-sm text-center">  
                    <div className="text-4xl mb-3">📄</div>  
                    <h3 className="text-lg font-bold text-slate-800">No PDF Loaded</h3>  
                    <p className="text-sm text-slate-500 max-w-md mx-auto mt-1">  
                      Upload your daily hospital floor stock delivery note PDF to separate items into their respective runs.  
                    </p>  
                  </div>  
                )  
              )}  
            </div>  
          ) : (  
            /* Admin & Item Category Management Settings */  
            <div className="bg-white rounded-xl shadow-sm p-6">  
              <h2 className="text-xl font-bold text-slate-800 mb-2">Category Rule Settings</h2>  
              <p className="text-xs text-slate-500 mb-6">Manage keywords used to automatically route incoming PDF items into the 6 category runs.</p>  
  
              {/* Add New Item Rule Form */}  
              <form onSubmit={handleAddItemRule} className="bg-slate-50 p-4 rounded-xl border border-slate-200 mb-8 flex flex-col md:flex-row gap-3 items-end">  
                <div className="flex-1 w-full">  
                  <label className="block text-xs font-semibold text-slate-600 mb-1">Item Name / Keyword</label>  
                  <input   
                    type="text"   
                    value={newCategoryItem}  
                    onChange={(e) => setNewCategoryItem(e.target.value)}  
                    placeholder="e.g., Apple Juice PC"  
                    className="w-full p-2.5 border rounded-lg text-sm focus:outline-emerald-500"  
                  />  
                </div>  
  
                <div className="w-full md:w-48">  
                  <label className="block text-xs font-semibold text-slate-600 mb-1">Assign to Category Run</label>  
                  <select   
                    value={targetCategory}  
                    onChange={(e) => setTargetCategory(e.target.value)}  
                    className="w-full p-2.5 border rounded-lg text-sm focus:outline-emerald-500 bg-white"  
                  >  
                    <option value="Juice Run">Juice Run</option>  
                    <option value="Chiller Run">Chiller Run</option>  
                    <option value="Dry Run">Dry Run</option>  
                    <option value="Freezer Run">Freezer Run</option>  
                    <option value="Afternoon Run">Afternoon Run</option>  
                  </select>  
                </div>  
  
                <button   
                  type="submit"  
                  className="w-full md:w-auto px-5 py-2.5 bg-emerald-600 hover:bg-emerald-700 text-white font-medium text-sm rounded-lg transition"  
                >  
                  Add Item Rule  
                </button>  
              </form>  
  
              {/* Rules List by Category */}  
              <div className="grid md:grid-cols-2 gap-6">  
                {Object.entries(rules).map(([category, keywords]) => (  
                  <div key={category} className="border border-slate-200 rounded-xl p-4">  
                    <h3 className="font-bold text-slate-800 border-b pb-2 mb-3 flex justify-between items-center">  
                      <span>{category}</span>  
                      <span className="text-xs bg-slate-100 px-2 py-0.5 rounded text-slate-600">{keywords.length} keywords</span>  
                    </h3>  
  
                    <div className="flex flex-wrap gap-1.5">  
                      {keywords.map((kw, idx) => (  
                        <span key={idx} className="bg-slate-100 text-slate-700 text-xs px-2.5 py-1 rounded-md flex items-center gap-2 border border-slate-200">  
                          {kw}  
                          <button   
                            onClick={() => handleRemoveRule(category, kw)}  
                            className="text-slate-400 hover:text-red-500 font-bold"  
                          >  
                            ×  
                          </button>  
                        </span>  
                      ))}  
                    </div>  
                  </div>  
                ))}  
              </div>  
            </div>  
          )}  
        </div>  
      );  
    }  
  
    const root = ReactDOM.createRoot(document.getElementById('root'));  
    root.render(<App />);  
  </script>  
</body>  
</html>  
