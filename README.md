import React, { useState, useEffect, useMemo, useRef } from 'react';
import { 
  Save, Printer, Settings, Plus, Trash2, 
  ChevronDown, ChevronUp, Layout, FileText, 
  User, Check, AlertCircle, Calculator, Send
} from 'lucide-react';
import { initializeApp } from 'firebase/app';
import { 
  getAuth, 
  signInAnonymously, 
  onAuthStateChanged,
  signInWithCustomToken
} from 'firebase/auth';
import { 
  getFirestore, 
  collection, 
  doc, 
  setDoc, 
  getDocs, 
  query, 
  serverTimestamp 
} from 'firebase/firestore';

// --- FIREBASE SETUP ---
const firebaseConfig = JSON.parse(__firebase_config || '{}');
const app = initializeApp(firebaseConfig);
const auth = getAuth(app);
const db = getFirestore(app);
const appId = typeof __app_id !== 'undefined' ? __app_id : 'urbanzen-v1';

// --- CONSTANTS & DEFAULTS ---

const DEFAULT_RATES = {
  materials: {
    'BWR Plywood': 120,
    'BWP Plywood': 150,
    'HDHMR': 110,
    'MDF': 75,
    'Particle Board': 55
  },
  finishes: {
    'Laminate (1mm)': 60,
    'Laminate (Premium)': 120,
    'Acrylic (High Gloss)': 250,
    'PU Polish': 350,
    'Duco Paint': 300,
    'Veneer': 180
  },
  hardwareMultipliers: {
    'Standard': 1.0,
    'Premium (Hettich/Hafele)': 1.4,
    'Luxury (Blum)': 2.2
  },
  laborRateSqFt: 150,
  taxRate: 18, // GST %
  designFeeFixed: 0 // Optional fixed fee
};

const ROOM_TEMPLATES = [
  { id: 'living', name: 'Living Room', icon: '🛋️' },
  { id: 'master', name: 'Master Bedroom', icon: '🛏️' },
  { id: 'kitchen', name: 'Kitchen', icon: '🍳' },
  { id: 'dining', name: 'Dining', icon: '🍽️' },
  { id: 'bath', name: 'Bathroom', icon: '🚿' },
  { id: 'custom', name: 'Custom Room', icon: '✨' }
];

const ITEM_TEMPLATES = [
  { name: 'Wardrobe', defaultW: 6, defaultH: 7, defaultD: 2, category: 'Joinery' },
  { name: 'Modular Kitchen Base', defaultW: 10, defaultH: 2.5, defaultD: 2, category: 'Kitchen' },
  { name: 'Modular Kitchen Overhead', defaultW: 10, defaultH: 2, defaultD: 1, category: 'Kitchen' },
  { name: 'TV Unit', defaultW: 6, defaultH: 5, defaultD: 1.5, category: 'Joinery' },
  { name: 'Crockery Unit', defaultW: 4, defaultH: 7, defaultD: 1.5, category: 'Joinery' },
  { name: 'Study Table', defaultW: 4, defaultH: 2.5, defaultD: 2, category: 'Joinery' },
  { name: 'False Ceiling', defaultW: 12, defaultH: 14, defaultD: 0, category: 'Civil' },
  { name: 'Wall Paneling', defaultW: 10, defaultH: 9, defaultD: 0, category: 'Civil' },
];

// --- UTILS ---

const formatCurrency = (amount) => {
  return new Intl.NumberFormat('en-IN', {
    style: 'currency',
    currency: 'INR',
    maximumFractionDigits: 0
  }).format(amount);
};

// --- COMPONENTS ---

// 1. Logo Component
const UrbanZenLogo = ({ size = 'md', dark = false }) => (
  <div className={`font-serif tracking-widest flex items-center gap-2 ${dark ? 'text-slate-900' : 'text-slate-50'}`}>
    <div className={`border-2 ${dark ? 'border-slate-900' : 'border-slate-50'} p-1`}>
      <span className={`font-bold ${size === 'lg' ? 'text-2xl' : 'text-xl'}`}>UZ</span>
    </div>
    <div className="flex flex-col leading-none">
      <span className={`font-bold ${size === 'lg' ? 'text-2xl' : 'text-lg'}`}>URBANZEN</span>
      <span className={`text-[10px] uppercase tracking-[0.3em] opacity-80`}>Interiors</span>
    </div>
  </div>
);

// 2. Main App Component
export default function App() {
  const [user, setUser] = useState(null);
  const [loading, setLoading] = useState(true);
  const [step, setStep] = useState(1);
  const [showSettings, setShowSettings] = useState(false);
  const [saveStatus, setSaveStatus] = useState('idle'); // idle, saving, saved, error

  // Core Data State
  const [client, setClient] = useState({
    name: '', phone: '', email: '', address: '', type: '3BHK', area: ''
  });
  
  const [rates, setRates] = useState(DEFAULT_RATES);
  
  const [projectRooms, setProjectRooms] = useState([]);

  // --- AUTH & LOAD ---
  useEffect(() => {
    const initAuth = async () => {
      if (typeof __initial_auth_token !== 'undefined' && __initial_auth_token) {
        await signInWithCustomToken(auth, __initial_auth_token);
      } else {
        await signInAnonymously(auth);
      }
    };
    initAuth();
    const unsubscribe = onAuthStateChanged(auth, setUser);
    setLoading(false);
    return () => unsubscribe();
  }, []);

  // --- ACTIONS ---

  const addRoom = (template) => {
    const newRoom = {
      id: crypto.randomUUID(),
      name: template.id === 'custom' ? 'New Room' : template.name,
      type: template.id,
      items: []
    };
    setProjectRooms([...projectRooms, newRoom]);
  };

  const removeRoom = (roomId) => {
    setProjectRooms(projectRooms.filter(r => r.id !== roomId));
  };

  const addItemToRoom = (roomId, itemTemplate) => {
    setProjectRooms(rooms => rooms.map(room => {
      if (room.id !== roomId) return room;
      return {
        ...room,
        items: [...room.items, {
          id: crypto.randomUUID(),
          name: itemTemplate.name,
          w: itemTemplate.defaultW,
          h: itemTemplate.defaultH,
          d: itemTemplate.defaultD,
          material: 'BWR Plywood',
          finish: 'Laminate (1mm)',
          hardware: 'Standard',
          qty: 1
        }]
      };
    }));
  };

  const updateItem = (roomId, itemId, field, value) => {
    setProjectRooms(rooms => rooms.map(room => {
      if (room.id !== roomId) return room;
      return {
        ...room,
        items: room.items.map(item => {
          if (item.id !== itemId) return item;
          return { ...item, [field]: value };
        })
      };
    }));
  };

  const deleteItem = (roomId, itemId) => {
    setProjectRooms(rooms => rooms.map(room => {
      if (room.id !== roomId) return room;
      return {
        ...room,
        items: room.items.filter(i => i.id !== itemId)
      };
    }));
  };

  // --- CALCULATION ENGINE ---
  
  const calculateItemPrice = (item) => {
    const sqFt = item.w * item.h;
    if (sqFt <= 0) return 0;

    const materialRate = rates.materials[item.material] || 0;
    const finishRate = rates.finishes[item.finish] || 0;
    const labor = rates.laborRateSqFt;
    
    // Base Cost per SqFt = (Material + Finish + Labor)
    let baseRate = materialRate + finishRate + labor;
    
    // Hardware Multiplier applies to the total base construction cost
    const hwMult = rates.hardwareMultipliers[item.hardware] || 1;
    
    // Total Unit Cost
    const unitCost = (baseRate * sqFt * hwMult);
    
    return unitCost * item.qty;
  };

  const calculateTotals = () => {
    let subtotal = 0;
    const roomTotals = projectRooms.map(room => {
      const roomTotal = room.items.reduce((sum, item) => sum + calculateItemPrice(item), 0);
      subtotal += roomTotal;
      return { id: room.id, total: roomTotal };
    });

    const tax = subtotal * (rates.taxRate / 100);
    const grandTotal = subtotal + tax + rates.designFeeFixed;

    return { subtotal, tax, grandTotal, roomTotals };
  };

  const totals = useMemo(() => calculateTotals(), [projectRooms, rates]);

  // --- DATABASE SAVING ---

  const saveProject = async () => {
    if (!user) return;
    setSaveStatus('saving');
    try {
      const projectData = {
        client,
        rooms: projectRooms,
        rates, // Save the rates snapshot used for this quote
        totals,
        updatedAt: serverTimestamp(),
        status: 'draft'
      };

      const docRef = doc(collection(db, 'artifacts', appId, 'users', user.uid, 'quotations'));
      await setDoc(docRef, projectData);
      setSaveStatus('saved');
      setTimeout(() => setSaveStatus('idle'), 3000);
    } catch (error) {
      console.error("Error saving:", error);
      setSaveStatus('error');
    }
  };

  // --- RENDER HELPERS ---

  if (loading) return <div className="h-screen flex items-center justify-center text-slate-500">Loading URBANZEN...</div>;

  return (
    <div className="min-h-screen bg-slate-100 font-sans text-slate-800 print:bg-white">
      
      {/* HEADER (No Print) */}
      <header className="bg-slate-900 text-white p-4 shadow-lg print:hidden sticky top-0 z-50">
        <div className="max-w-7xl mx-auto flex justify-between items-center">
          <UrbanZenLogo />
          <div className="flex items-center gap-4">
            <span className="hidden md:inline text-slate-400 text-sm">
              {client.name ? `Project: ${client.name}` : 'New Project'}
            </span>
            <button 
              onClick={() => setShowSettings(true)}
              className="p-2 hover:bg-slate-800 rounded-full transition-colors"
              title="Rate Card Settings"
            >
              <Settings size={20} />
            </button>
            <div className="flex gap-2">
               <button 
                onClick={() => setStep(1)}
                className={`px-3 py-1 rounded text-sm ${step === 1 ? 'bg-amber-600' : 'hover:bg-slate-800'}`}
              >Client</button>
               <button 
                onClick={() => setStep(2)}
                disabled={!client.name}
                className={`px-3 py-1 rounded text-sm ${step === 2 ? 'bg-amber-600' : 'hover:bg-slate-800 disabled:opacity-30'}`}
              >Design</button>
               <button 
                onClick={() => setStep(3)}
                disabled={projectRooms.length === 0}
                className={`px-3 py-1 rounded text-sm ${step === 3 ? 'bg-amber-600' : 'hover:bg-slate-800 disabled:opacity-30'}`}
              >Preview</button>
            </div>
          </div>
        </div>
      </header>

      {/* MAIN CONTENT */}
      <main className="max-w-7xl mx-auto p-4 md:p-8 print:p-0">
        
        {/* STEP 1: CLIENT DETAILS */}
        {step === 1 && (
          <div className="max-w-2xl mx-auto bg-white rounded-xl shadow-sm p-8 animate-in fade-in slide-in-from-bottom-4 duration-500">
            <h2 className="text-2xl font-bold text-slate-800 mb-6 flex items-center gap-2">
              <User className="text-amber-600" /> Client Details
            </h2>
            <div className="grid grid-cols-1 md:grid-cols-2 gap-6">
              <div className="space-y-2">
                <label className="text-sm font-medium text-slate-600">Client Name</label>
                <input 
                  type="text" 
                  value={client.name}
                  onChange={e => setClient({...client, name: e.target.value})}
                  className="w-full p-3 border border-slate-200 rounded-lg focus:ring-2 focus:ring-amber-500 outline-none"
                  placeholder="e.g. Rahul Sharma"
                />
              </div>
              <div className="space-y-2">
                <label className="text-sm font-medium text-slate-600">Phone Number</label>
                <input 
                  type="tel" 
                  value={client.phone}
                  onChange={e => setClient({...client, phone: e.target.value})}
                  className="w-full p-3 border border-slate-200 rounded-lg focus:ring-2 focus:ring-amber-500 outline-none"
                  placeholder="+91 98765 43210"
                />
              </div>
              <div className="space-y-2 md:col-span-2">
                <label className="text-sm font-medium text-slate-600">Email Address</label>
                <input 
                  type="email" 
                  value={client.email}
                  onChange={e => setClient({...client, email: e.target.value})}
                  className="w-full p-3 border border-slate-200 rounded-lg focus:ring-2 focus:ring-amber-500 outline-none"
                  placeholder="client@example.com"
                />
              </div>
              <div className="space-y-2 md:col-span-2">
                <label className="text-sm font-medium text-slate-600">Project Address</label>
                <textarea 
                  value={client.address}
                  onChange={e => setClient({...client, address: e.target.value})}
                  className="w-full p-3 border border-slate-200 rounded-lg focus:ring-2 focus:ring-amber-500 outline-none"
                  placeholder="Complete address..."
                  rows={2}
                />
              </div>
              <div className="space-y-2">
                <label className="text-sm font-medium text-slate-600">Property Type</label>
                <select 
                  value={client.type}
                  onChange={e => setClient({...client, type: e.target.value})}
                  className="w-full p-3 border border-slate-200 rounded-lg focus:ring-2 focus:ring-amber-500 outline-none"
                >
                  <option>1BHK</option>
                  <option>2BHK</option>
                  <option>3BHK</option>
                  <option>Villa</option>
                  <option>Office</option>
                </select>
              </div>
              <div className="space-y-2">
                <label className="text-sm font-medium text-slate-600">Carpet Area (Sq.Ft)</label>
                <input 
                  type="number" 
                  value={client.area}
                  onChange={e => setClient({...client, area: e.target.value})}
                  className="w-full p-3 border border-slate-200 rounded-lg focus:ring-2 focus:ring-amber-500 outline-none"
                  placeholder="e.g. 1200"
                />
              </div>
            </div>
            <div className="mt-8 flex justify-end">
              <button 
                onClick={() => setStep(2)}
                disabled={!client.name}
                className="bg-slate-900 text-white px-6 py-3 rounded-lg hover:bg-slate-800 flex items-center gap-2 disabled:opacity-50 transition-all"
              >
                Start Quotation <ChevronDown className="-rotate-90" size={18} />
              </button>
            </div>
          </div>
        )}

        {/* STEP 2: BUILDER */}
        {step === 2 && (
          <div className="grid grid-cols-1 lg:grid-cols-12 gap-6">
            
            {/* Left: Room Selector */}
            <div className="lg:col-span-3 space-y-4">
              <div className="bg-white p-4 rounded-xl shadow-sm">
                <h3 className="font-bold text-slate-800 mb-4">Add Room</h3>
                <div className="grid grid-cols-2 gap-2">
                  {ROOM_TEMPLATES.map(template => (
                    <button
                      key={template.id}
                      onClick={() => addRoom(template)}
                      className="flex flex-col items-center justify-center p-3 border border-slate-100 rounded hover:bg-amber-50 hover:border-amber-200 transition-colors"
                    >
                      <span className="text-2xl mb-1">{template.icon}</span>
                      <span className="text-xs text-center">{template.name}</span>
                    </button>
                  ))}
                </div>
              </div>
              
              <div className="bg-slate-900 text-white p-6 rounded-xl shadow-sm sticky top-24">
                <h3 className="text-amber-500 font-medium text-sm uppercase tracking-wider mb-2">Estimated Cost</h3>
                <div className="text-3xl font-bold mb-4">{formatCurrency(totals.grandTotal)}</div>
                <div className="space-y-2 text-sm opacity-80 border-t border-slate-700 pt-4">
                  <div className="flex justify-between">
                    <span>Subtotal</span>
                    <span>{formatCurrency(totals.subtotal)}</span>
                  </div>
                  <div className="flex justify-between">
                    <span>GST ({rates.taxRate}%)</span>
                    <span>{formatCurrency(totals.tax)}</span>
                  </div>
                </div>
                <button 
                  onClick={() => setStep(3)}
                  disabled={projectRooms.length === 0}
                  className="w-full mt-6 bg-amber-600 hover:bg-amber-700 text-white py-3 rounded font-medium transition-colors disabled:opacity-50"
                >
                  Generate Quote
                </button>
              </div>
            </div>

            {/* Right: Configuration */}
            <div className="lg:col-span-9 space-y-6">
              {projectRooms.length === 0 ? (
                <div className="h-64 flex flex-col items-center justify-center text-slate-400 border-2 border-dashed border-slate-300 rounded-xl">
                  <Layout size={48} className="mb-4 opacity-50" />
                  <p>Select a room type to start adding items.</p>
                </div>
              ) : (
                projectRooms.map((room, rIndex) => (
                  <div key={room.id} className="bg-white rounded-xl shadow-sm overflow-hidden border border-slate-100 animate-in fade-in slide-in-from-bottom-2">
                    <div className="bg-slate-50 p-4 border-b border-slate-100 flex justify-between items-center">
                      <div className="flex items-center gap-3">
                         <span className="text-2xl">{ROOM_TEMPLATES.find(t => t.id === room.type)?.icon}</span>
                         <input 
                           value={room.name}
                           onChange={(e) => {
                             const newRooms = [...projectRooms];
                             newRooms[rIndex].name = e.target.value;
                             setProjectRooms(newRooms);
                           }}
                           className="font-bold text-lg bg-transparent border-b border-transparent hover:border-slate-300 focus:border-amber-500 outline-none"
                         />
                         <span className="text-sm bg-slate-200 px-2 py-0.5 rounded text-slate-600">
                           {formatCurrency(totals.roomTotals.find(t => t.id === room.id)?.total || 0)}
                         </span>
                      </div>
                      <button onClick={() => removeRoom(room.id)} className="text-slate-400 hover:text-red-500">
                        <Trash2 size={18} />
                      </button>
                    </div>

                    <div className="p-4 space-y-4">
                      {room.items.length === 0 && (
                         <div className="text-center py-4 text-slate-400 text-sm">No items in this room yet.</div>
                      )}
                      
                      {room.items.map(item => (
                        <div key={item.id} className="grid grid-cols-1 md:grid-cols-12 gap-4 items-end bg-slate-50 p-4 rounded-lg border border-slate-100 relative group">
                          
                          {/* Row 1: Item Details */}
                          <div className="md:col-span-3">
                             <label className="text-xs text-slate-500 mb-1 block">Item</label>
                             <input 
                               value={item.name} 
                               onChange={e => updateItem(room.id, item.id, 'name', e.target.value)}
                               className="w-full text-sm font-medium bg-transparent border-b border-slate-300 focus:border-amber-500 outline-none"
                             />
                          </div>

                          <div className="md:col-span-3 grid grid-cols-3 gap-2">
                            <div>
                               <label className="text-xs text-slate-500 mb-1 block">W (ft)</label>
                               <input type="number" value={item.w} onChange={e => updateItem(room.id, item.id, 'w', parseFloat(e.target.value))} className="w-full p-1 text-sm border rounded" />
                            </div>
                            <div>
                               <label className="text-xs text-slate-500 mb-1 block">H (ft)</label>
                               <input type="number" value={item.h} onChange={e => updateItem(room.id, item.id, 'h', parseFloat(e.target.value))} className="w-full p-1 text-sm border rounded" />
                            </div>
                            <div>
                               <label className="text-xs text-slate-500 mb-1 block">Qty</label>
                               <input type="number" value={item.qty} onChange={e => updateItem(room.id, item.id, 'qty', parseFloat(e.target.value))} className="w-full p-1 text-sm border rounded" />
                            </div>
                          </div>

                          {/* Row 2: Specs */}
                          <div className="md:col-span-2">
                            <label className="text-xs text-slate-500 mb-1 block">Material</label>
                            <select 
                              value={item.material}
                              onChange={e => updateItem(room.id, item.id, 'material', e.target.value)}
                              className="w-full text-xs p-1.5 border rounded"
                            >
                              {Object.keys(rates.materials).map(k => <option key={k} value={k}>{k}</option>)}
                            </select>
                          </div>

                          <div className="md:col-span-2">
                            <label className="text-xs text-slate-500 mb-1 block">Finish</label>
                            <select 
                              value={item.finish}
                              onChange={e => updateItem(room.id, item.id, 'finish', e.target.value)}
                              className="w-full text-xs p-1.5 border rounded"
                            >
                              {Object.keys(rates.finishes).map(k => <option key={k} value={k}>{k}</option>)}
                            </select>
                          </div>
                          
                          <div className="md:col-span-2 text-right">
                             <div className="text-xs text-slate-400">Total</div>
                             <div className="font-bold text-slate-800">{formatCurrency(calculateItemPrice(item))}</div>
                          </div>

                          <button 
                            onClick={() => deleteItem(room.id, item.id)}
                            className="absolute top-2 right-2 text-slate-300 hover:text-red-500 opacity-0 group-hover:opacity-100 transition-opacity"
                          >
                            <Trash2 size={14} />
                          </button>
                        </div>
                      ))}

                      {/* Add Item Bar */}
                      <div className="flex flex-wrap gap-2 mt-4 pt-4 border-t border-slate-100">
                         {ITEM_TEMPLATES.map(t => (
                           <button 
                            key={t.name}
                            onClick={() => addItemToRoom(room.id, t)}
                            className="text-xs bg-white border border-slate-200 hover:border-amber-500 hover:text-amber-600 px-3 py-1.5 rounded-full transition-colors flex items-center gap-1"
                           >
                             <Plus size={12} /> {t.name}
                           </button>
                         ))}
                      </div>
                    </div>
                  </div>
                ))
              )}
            </div>
          </div>
        )}

        {/* STEP 3: PREVIEW (PRINT AREA) */}
        {step === 3 && (
          <div className="flex flex-col items-center">
            
            {/* Action Bar */}
            <div className="w-full max-w-[210mm] mb-6 flex justify-between items-center bg-white p-4 rounded-lg shadow-sm print:hidden">
              <div className="flex gap-4">
                 <button 
                  onClick={() => window.print()}
                  className="bg-slate-900 text-white px-4 py-2 rounded flex items-center gap-2 hover:bg-slate-800"
                 >
                   <Printer size={18} /> Print / Download PDF
                 </button>
                 <button 
                  onClick={saveProject}
                  disabled={saveStatus === 'saving'}
                  className="bg-white border border-slate-200 text-slate-700 px-4 py-2 rounded flex items-center gap-2 hover:bg-slate-50"
                 >
                   {saveStatus === 'saving' ? 'Saving...' : saveStatus === 'saved' ? 'Saved!' : 'Save Online'}
                   {saveStatus === 'saved' && <Check size={18} className="text-green-500" />}
                 </button>
              </div>
              <div className="text-sm text-slate-500">
                Ensure "Background Graphics" is ON in print settings
              </div>
            </div>

            {/* A4 Paper Simulation */}
            <div className="bg-white shadow-2xl print:shadow-none w-full max-w-[210mm] min-h-[297mm] p-[10mm] md:p-[15mm] text-slate-900 relative">
              
              {/* PDF Header */}
              <div className="flex justify-between items-start border-b-2 border-slate-900 pb-6 mb-8">
                 <UrbanZenLogo size="lg" dark />
                 <div className="text-right">
                    <h1 className="text-4xl font-light text-slate-300 uppercase tracking-widest mb-2">Quotation</h1>
                    <div className="text-sm font-bold text-slate-800">#{Math.floor(Date.now() / 1000)}</div>
                    <div className="text-sm text-slate-500">{new Date().toLocaleDateString()}</div>
                 </div>
              </div>

              {/* Client Info Grid */}
              <div className="flex justify-between mb-10 text-sm">
                <div>
                   <h3 className="text-xs font-bold uppercase tracking-wider text-slate-400 mb-2">Bill To</h3>
                   <div className="font-bold text-lg">{client.name}</div>
                   <div>{client.address}</div>
                   <div>{client.phone}</div>
                   <div>{client.email}</div>
                </div>
                <div className="text-right">
                   <h3 className="text-xs font-bold uppercase tracking-wider text-slate-400 mb-2">Project Details</h3>
                   <div className="font-bold">{client.type} ({client.area} Sq.Ft)</div>
                   <div className="text-amber-600 font-medium">Valid until: {new Date(Date.now() + 15 * 86400000).toLocaleDateString()}</div>
                </div>
              </div>

              {/* Items Table */}
              <div className="mb-8">
                <table className="w-full text-sm">
                  <thead>
                    <tr className="border-b-2 border-slate-800 text-left">
                      <th className="py-2 font-bold w-12">#</th>
                      <th className="py-2 font-bold">Item Description</th>
                      <th className="py-2 font-bold">Spec</th>
                      <th className="py-2 font-bold text-center">Qty</th>
                      <th className="py-2 font-bold text-right">Amount</th>
                    </tr>
                  </thead>
                  <tbody className="divide-y divide-slate-100">
                    {projectRooms.map((room) => (
                      <React.Fragment key={room.id}>
                        {/* Room Header Row */}
                        <tr className="bg-slate-50 print:bg-slate-50">
                          <td colSpan="5" className="py-2 px-2 font-bold text-slate-700 italic border-l-4 border-amber-500">
                             {room.name}
                          </td>
                        </tr>
                        {/* Items */}
                        {room.items.map((item, idx) => (
                          <tr key={item.id}>
                            <td className="py-3 text-slate-400">{idx + 1}</td>
                            <td className="py-3">
                              <div className="font-medium text-slate-800">{item.name}</div>
                              <div className="text-xs text-slate-500">
                                {item.w}' x {item.h}' | {item.material}
                              </div>
                            </td>
                            <td className="py-3 text-xs text-slate-600">
                              <div>{item.finish}</div>
                              <div>{item.hardware} Hardware</div>
                            </td>
                            <td className="py-3 text-center">{item.qty}</td>
                            <td className="py-3 text-right font-medium">
                              {formatCurrency(calculateItemPrice(item))}
                            </td>
                          </tr>
                        ))}
                      </React.Fragment>
                    ))}
                  </tbody>
                </table>
              </div>

              {/* Summary */}
              <div className="flex justify-end mb-12 break-inside-avoid">
                 <div className="w-64 space-y-2">
                    <div className="flex justify-between text-sm">
                       <span className="text-slate-500">Subtotal</span>
                       <span className="font-medium">{formatCurrency(totals.subtotal)}</span>
                    </div>
                    {rates.designFeeFixed > 0 && (
                      <div className="flex justify-between text-sm">
                         <span className="text-slate-500">Design Fees</span>
                         <span className="font-medium">{formatCurrency(rates.designFeeFixed)}</span>
                      </div>
                    )}
                    <div className="flex justify-between text-sm">
                       <span className="text-slate-500">GST ({rates.taxRate}%)</span>
                       <span className="font-medium">{formatCurrency(totals.tax)}</span>
                    </div>
                    <div className="flex justify-between text-lg font-bold border-t-2 border-slate-900 pt-2 mt-2">
                       <span>Total</span>
                       <span className="text-amber-600">{formatCurrency(totals.grandTotal)}</span>
                    </div>
                 </div>
              </div>

              {/* Terms */}
              <div className="border-t border-slate-200 pt-8 mt-auto break-inside-avoid">
                <div className="grid grid-cols-2 gap-8 text-xs text-slate-500">
                   <div>
                      <h4 className="font-bold text-slate-800 mb-2">Terms & Conditions</h4>
                      <ul className="list-disc list-inside space-y-1">
                        <li>50% Advance payment to commence work.</li>
                        <li>40% Payment before material delivery.</li>
                        <li>10% on handover.</li>
                        <li>Civil works and electrical appliances not included unless specified.</li>
                      </ul>
                   </div>
                   <div className="text-right flex flex-col justify-end">
                      <div className="h-16 mb-2"></div> 
                      <div className="font-bold text-slate-800">Authorized Signatory</div>
                      <div>URBANZEN INTERIORS</div>
                   </div>
                </div>
              </div>

            </div>
          </div>
        )}

      </main>

      {/* SETTINGS MODAL */}
      {showSettings && (
        <div className="fixed inset-0 bg-black/50 z-[60] flex items-center justify-center p-4 print:hidden">
          <div className="bg-white rounded-xl shadow-2xl w-full max-w-2xl max-h-[90vh] overflow-y-auto">
             <div className="p-6 border-b border-slate-100 flex justify-between items-center bg-slate-50">
                <h2 className="text-xl font-bold flex items-center gap-2">
                  <Calculator size={24} className="text-amber-600" /> Rate Card Configuration
                </h2>
                <button onClick={() => setShowSettings(false)} className="text-slate-400 hover:text-slate-600">
                  <Plus className="rotate-45" size={24} />
                </button>
             </div>
             
             <div className="p-6 space-y-8">
                <section>
                   <h3 className="font-bold text-slate-800 mb-4 border-b pb-2">Material Base Rates (₹/sq.ft)</h3>
                   <div className="grid grid-cols-2 gap-4">
                      {Object.entries(rates.materials).map(([key, val]) => (
                        <div key={key} className="flex justify-between items-center text-sm">
                           <span>{key}</span>
                           <input 
                            type="number" 
                            value={val}
                            onChange={e => setRates({...rates, materials: {...rates.materials, [key]: parseFloat(e.target.value)}})}
                            className="w-20 p-1 border rounded text-right"
                           />
                        </div>
                      ))}
                   </div>
                </section>

                <section>
                   <h3 className="font-bold text-slate-800 mb-4 border-b pb-2">Finish Rates (Add-on ₹/sq.ft)</h3>
                   <div className="grid grid-cols-2 gap-4">
                      {Object.entries(rates.finishes).map(([key, val]) => (
                        <div key={key} className="flex justify-between items-center text-sm">
                           <span>{key}</span>
                           <input 
                            type="number" 
                            value={val}
                            onChange={e => setRates({...rates, finishes: {...rates.finishes, [key]: parseFloat(e.target.value)}})}
                            className="w-20 p-1 border rounded text-right"
                           />
                        </div>
                      ))}
                   </div>
                </section>

                <section>
                   <h3 className="font-bold text-slate-800 mb-4 border-b pb-2">Global Settings</h3>
                   <div className="grid grid-cols-2 gap-4">
                      <div className="flex justify-between items-center text-sm">
                         <span>Labor Rate (₹/sq.ft)</span>
                         <input 
                          type="number" 
                          value={rates.laborRateSqFt}
                          onChange={e => setRates({...rates, laborRateSqFt: parseFloat(e.target.value)})}
                          className="w-20 p-1 border rounded text-right"
                         />
                      </div>
                      <div className="flex justify-between items-center text-sm">
                         <span>GST (%)</span>
                         <input 
                          type="number" 
                          value={rates.taxRate}
                          onChange={e => setRates({...rates, taxRate: parseFloat(e.target.value)})}
                          className="w-20 p-1 border rounded text-right"
                         />
                      </div>
                   </div>
                </section>
             </div>
             <div className="p-6 bg-slate-50 border-t border-slate-100 flex justify-end">
                <button 
                  onClick={() => setShowSettings(false)}
                  className="bg-slate-900 text-white px-6 py-2 rounded hover:bg-slate-800"
                >
                  Save & Close
                </button>
             </div>
          </div>
        </div>
      )}
    </div>
  );
}
