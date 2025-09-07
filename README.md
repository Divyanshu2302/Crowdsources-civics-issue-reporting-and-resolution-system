# Crowdsources-civics-issue-reporting-and-resolution-system
import React, { useState, useEffect, useCallback, useRef } from 'react';

// --- Reusable SVG Icons ---
const MapPinIcon = ({ className }) => (
    <svg xmlns="http://www.w3.org/2000/svg" width="24" height="24" viewBox="0 0 24 24" fill="none" stroke="currentColor" strokeWidth="2" strokeLinecap="round" strokeLinejoin="round" className={className}>
        <path d="M21 10c0 7-9 13-9 13s-9-6-9-13a9 9 0 0 1 18 0z"></path>
        <circle cx="12" cy="10" r="3"></circle>
    </svg>
);

const EditIcon = ({ className }) => (
    <svg xmlns="http://www.w3.org/2000/svg" width="24" height="24" viewBox="0 0 24 24" fill="none" stroke="currentColor" strokeWidth="2" strokeLinecap="round" strokeLinejoin="round" className={className}>
        <path d="M11 4H4a2 2 0 0 0-2 2v14a2 2 0 0 0 2 2h14a2 2 0 0 0 2-2v-7"></path>
        <path d="M18.5 2.5a2.121 2.121 0 0 1 3 3L12 15l-4 1 1-4 9.5-9.5z"></path>
    </svg>
);

const CheckCircleIcon = ({ className }) => (
     <svg xmlns="http://www.w3.org/2000/svg" width="24" height="24" viewBox="0 0 24 24" fill="none" stroke="currentColor" strokeWidth="2" strokeLinecap="round" strokeLinejoin="round" className={className}>
        <path d="M22 11.08V12a10 10 0 1 1-5.93-9.14"></path>
        <polyline points="22 4 12 14.01 9 11.01"></polyline>
    </svg>
);

// --- Initial Data with Latitude/Longitude for Delhi, India ---
const initialReports = [
    { id: 1, type: 'Waste Dumping', description: 'Overflowing bins near India Gate.', lat: 28.6129, lng: 77.2295, status: 'Reported' },
    { id: 2, type: 'Broken Streetlight', description: 'Light out near Connaught Place.', lat: 28.6330, lng: 77.2196, status: 'In Progress' },
    { id: 3, type: 'Pothole', description: 'Large pothole near Red Fort.', lat: 28.6562, lng: 77.2410, status: 'Resolved' }
];

// --- Sub-Components that don't depend on Leaflet ---

const Header = () => (
    <header className="bg-white shadow-sm p-4 z-10">
        <div className="container mx-auto text-center">
            <h1 className="text-3xl md:text-4xl font-bold header-gradient">Urban Greenscape Navigator</h1>
            <p className="text-slate-500 mt-1">Your Community's Pathway to a Sustainable Future</p>
        </div>
    </header>
);

const ReportForm = ({ onSubmit, onCancel }) => {
    const [formData, setFormData] = useState({
        'issue-type': 'Waste Dumping',
        description: '',
    });

    const handleChange = (e) => {
        const { name, value } = e.target;
        setFormData(prev => ({ ...prev, [name]: value }));
    };
    
    const handleSubmit = (e) => {
        e.preventDefault();
        if(!formData.description) {
            console.error("Description is required.");
            return;
        }
        onSubmit(formData);
    };

    return (
        <form onSubmit={handleSubmit} className="space-y-4">
            <h3 className="text-xl font-semibold">New Issue Report</h3>
            <div>
                <label htmlFor="issue-type" className="block text-sm font-medium text-slate-700">Type of Issue</label>
                <select id="issue-type" name="issue-type" value={formData['issue-type']} onChange={handleChange} className="mt-1 block w-full pl-3 pr-10 py-2 text-base border-slate-300 focus:outline-none focus:ring-green-500 focus:border-green-500 sm:text-sm rounded-md" required>
                    <option>Waste Dumping</option>
                    <option>Broken Streetlight</option>
                    <option>Overgrown Vegetation</option>
                    <option>Pothole</option>
                    <option>Vandalism</option>
                    <option>Other</option>
                </select>
            </div>
            <div>
                <label htmlFor="description" className="block text-sm font-medium text-slate-700">Description</label>
                <textarea id="description" name="description" rows="3" value={formData.description} onChange={handleChange} className="mt-1 shadow-sm focus:ring-green-500 focus:border-green-500 block w-full sm:text-sm border border-slate-300 rounded-md p-2" placeholder="e.g., Large pile of trash bags on the corner of..." required></textarea>
            </div>
            <div>
                <label htmlFor="photo" className="block text-sm font-medium text-slate-700">Upload Photo (Optional)</label>
                <input type="file" name="photo" id="photo" className="mt-1 block w-full text-sm text-slate-500 file:mr-4 file:py-2 file:px-4 file:rounded-full file:border-0 file:text-sm file:font-semibold file:bg-green-50 file:text-green-700 hover:file:bg-green-100"/>
            </div>
            <div className="flex space-x-2">
                 <button type="button" onClick={onCancel} className="w-1/2 bg-slate-200 text-slate-800 font-bold py-2 px-4 rounded-lg hover:bg-slate-300 transition-colors">Cancel</button>
                 <button type="submit" className="w-1/2 bg-green-600 text-white font-bold py-2 px-4 rounded-lg hover:bg-green-700 transition-colors">Submit Report</button>
            </div>
        </form>
    );
};

const SuccessToast = ({ isVisible }) => {
    if (!isVisible) return null;
    return (
        <div className="fixed bottom-5 right-5 bg-emerald-500 text-white py-3 px-6 rounded-lg shadow-xl flex items-center fade-in-out">
            <CheckCircleIcon className="mr-3" />
            <span className="font-semibold">Report submitted successfully! Thank you.</span>
        </div>
    );
};


// --- Main App Component ---
export default function App() {
    const StyleInjector = () => (
        <style>{`
            @import url("https://unpkg.com/leaflet@1.9.4/dist/leaflet.css");
            .fade-in-out { animation: fadeInOut 4s ease-in-out forwards; }
            @keyframes fadeInOut { 0%, 100% { opacity: 0; transform: translateY(20px); } 10%, 90% { opacity: 1; transform: translateY(0); } }
            .grow-subtle { transition: transform 0.3s cubic-bezier(0.25, 0.46, 0.45, 0.94); }
            .grow-subtle:hover { transform: scale(1.03); }
            .header-gradient { background: linear-gradient(to right, #16a34a, #059669); -webkit-background-clip: text; -webkit-text-fill-color: transparent; background-clip: text; text-fill-color: transparent; }
            .leaflet-marker-pulsing, .leaflet-marker-pulsing .leaflet-marker-shadow {
                animation: pulse 1.5s infinite;
            }
            @keyframes pulse {
                0% { transform: scale(1); opacity: 1; }
                50% { transform: scale(1.2); opacity: 0.7; }
                100% { transform: scale(1); opacity: 1; }
            }
        `}</style>
    );
    
    const [reports, setReports] = useState(initialReports);
    const [isFormVisible, setIsFormVisible] = useState(false);
    const [newPinLocation, setNewPinLocation] = useState(null);
    const [isToastVisible, setIsToastVisible] = useState(false);
    const [isLeafletReady, setIsLeafletReady] = useState(!!(window.ReactLeaflet && window.L));

    useEffect(() => {
        if (isToastVisible) {
            const timer = setTimeout(() => setIsToastVisible(false), 4000);
            return () => clearTimeout(timer);
        }
    }, [isToastVisible]);

    useEffect(() => {
        if (isLeafletReady) return;

        const interval = setInterval(() => {
            if (window.ReactLeaflet && window.L) {
                setIsLeafletReady(true);
                clearInterval(interval);
            }
        }, 100);

        const timeout = setTimeout(() => {
            clearInterval(interval);
        }, 5000); // Stop checking after 5 seconds if library fails to load

        return () => {
            clearInterval(interval);
            clearTimeout(timeout);
        };
    }, [isLeafletReady]);
    
    const handleMapClick = useCallback((location) => {
        setNewPinLocation(location);
        setIsFormVisible(true);
    }, []);

    const handleAddReport = (formData) => {
        if (!newPinLocation) {
            console.error("Please select a location on the map before submitting.");
            return;
        }
        const newReport = {
            id: Date.now(),
            type: formData['issue-type'],
            description: formData.description,
            lat: newPinLocation.lat,
            lng: newPinLocation.lng,
            status: 'Reported'
        };
        setReports(prev => [...prev, newReport]);
        setIsFormVisible(false);
        setNewPinLocation(null);
        setIsToastVisible(true);
    };
    
    const handleCancelForm = () => {
        setIsFormVisible(false);
        setNewPinLocation(null);
    };
    
    const handleShowForm = () => {
        setIsFormVisible(true);
        setNewPinLocation(null); 
    };

    if (!isLeafletReady) {
        return (
             <div className="min-h-screen flex flex-col justify-center items-center bg-slate-50">
                <Header />
                <div className="flex-grow flex justify-center items-center">
                    <p className="text-slate-500 animate-pulse text-lg">Loading Map Components...</p>
                </div>
            </div>
        )
    }

    // --- Leaflet Component ---
    // Defined inside App to ensure libraries (window.ReactLeaflet) are loaded before it's used.
    const { MapContainer, TileLayer, Marker, Popup, useMapEvents } = window.ReactLeaflet;

    const LeafletMapComponent = ({ reports, onMapClick, newPinLocation }) => {
        const MapClickHandler = () => {
            useMapEvents({
                click(e) { onMapClick(e.latlng); },
            });
            return null;
        };

        const newPinIcon = new L.Icon({
            iconUrl: 'https://cdnjs.cloudflare.com/ajax/libs/leaflet/1.7.1/images/marker-icon-2x.png',
            iconSize: [25, 41],
            iconAnchor: [12, 41],
            popupAnchor: [1, -34],
            shadowUrl: 'https://cdnjs.cloudflare.com/ajax/libs/leaflet/1.7.1/images/marker-shadow.png',
            shadowSize: [41, 41],
            className: 'leaflet-marker-pulsing'
        });

        return (
           <div className="lg:col-span-3 bg-white rounded-xl shadow-lg p-4 flex flex-col">
                <h2 className="text-xl font-semibold mb-3 flex items-center">
                    <MapPinIcon className="text-green-600 mr-2" />
                    Community Issue Map
                </h2>
                <MapContainer center={[28.6139, 77.2090]} zoom={12} className="w-full aspect-video rounded-lg z-0">
                    <TileLayer
                        attribution='&copy; <a href="https://www.openstreetmap.org/copyright">OpenStreetMap</a> contributors'
                        url="https://{s}.tile.openstreetmap.org/{z}/{x}/{y}.png"
                    />
                    <MapClickHandler />

                    {reports.map(report => (
                        <Marker key={report.id} position={[report.lat, report.lng]}>
                            <Popup>
                                <strong>{report.type} ({report.status})</strong><br />{report.description}
                            </Popup>
                        </Marker>
                    ))}

                    {newPinLocation && (
                        <Marker position={[newPinLocation.lat, newPinLocation.lng]} icon={newPinIcon}></Marker>
                    )}
                </MapContainer>
                <p className="text-sm text-slate-500 mt-2 text-center">Click on the map to report a new issue.</p>
            </div>
        );
    };

    return (
        <>
            <StyleInjector />
            <div className="min-h-screen flex flex-col bg-slate-50 font-sans text-slate-800">
                <Header />
                <main className="container mx-auto p-4 lg:p-8 flex-grow grid grid-cols-1 lg:grid-cols-5 gap-8">
                    
                    <LeafletMapComponent reports={reports} onMapClick={handleMapClick} newPinLocation={newPinLocation} />
                    
                    <div className="lg:col-span-2 bg-white rounded-xl shadow-lg p-6 flex flex-col">
                         {isFormVisible ? (
                            <ReportForm onSubmit={handleAddReport} onCancel={handleCancelForm} />
                         ) : (
                            <>
                                <div id="welcome-message">
                                    <h2 className="text-2xl font-bold mb-2">Help Build a Greener City!</h2>
                                    <p className="text-slate-600 mb-4">Click the map to pinpoint an issue, or tap the button below to describe a general problem.</p>
                                    <button onClick={handleShowForm} className="w-full bg-green-600 text-white font-bold py-3 px-4 rounded-lg hover:bg-green-700 transition-all duration-300 flex items-center justify-center text-lg grow-subtle">
                                         <EditIcon className="mr-2" />
                                        Report an Issue
                                    </button>
                                </div>
                                <hr className="my-6 border-slate-200" />
                                <div className="flex-grow flex flex-col min-h-0">
                                    <h3 className="text-xl font-semibold mb-3">Recently Reported</h3>
                                    <div id="reports-list" className="flex-grow overflow-y-auto pr-2 space-y-3">
                                        {[...reports].reverse().map(report => (
                                            <div key={report.id} className="p-3 rounded-lg border border-slate-200 grow-subtle">
                                                <div className="flex justify-between items-start">
                                                    <h4 className="font-semibold text-slate-800">{report.type}</h4>
                                                    <span className={`text-xs font-bold px-2 py-1 rounded-full ${
                                                        report.status === 'In Progress' ? 'bg-blue-100 text-blue-800' :
                                                        report.status === 'Resolved' ? 'bg-green-100 text-green-800' :
                                                        'bg-amber-100 text-amber-800'
                                                    }`}>
                                                        {report.status}
                                                    </span>
                                                </div>
                                                <p className="text-sm text-slate-600 mt-1">{report.description}</p>
                                            </div>
                                        ))}
                                    </div>
                                </div>
                            </>
                         )}
                    </div>
                </main>
                <SuccessToast isVisible={isToastVisible} />
            </div>
        </>
    );
}

