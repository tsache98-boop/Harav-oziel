# Harav-oziel
Project about buildings 
import React, { useState, useRef, useEffect } from 'react';
import { FileText, Users, Download, Send, CheckCircle, Clock, X, Upload, Eye, Copy, Link } from 'lucide-react';

const ApartmentSignatureApp = () => {
  const [view, setView] = useState('dashboard');
  const [signatures, setSignatures] = useState([]);
  const [currentApartment, setCurrentApartment] = useState(null);
  const [pdfData, setPdfData] = useState(null);
  const [showLinkModal, setShowLinkModal] = useState(false);
  const [selectedAptForLink, setSelectedAptForLink] = useState(null);
  const [formData, setFormData] = useState({
    fullName: '',
    date: new Date().toISOString().split('T')[0],
    email: '',
    phone: ''
  });
  const canvasRef = useRef(null);
  const fileInputRef = useRef(null);
  const [isDrawing, setIsDrawing] = useState(false);
  const [hasSignature, setHasSignature] = useState(false);

  useEffect(() => {
    loadSignatures();
    loadPDF();
    const params = new URLSearchParams(window.location.search);
    const aptParam = params.get('apt');
    if (aptParam) {
      const aptId = parseInt(aptParam);
      if (aptId >= 1 && aptId <= 32) {
        setCurrentApartment(aptId);
        setView('sign');
      }
    }
  }, []);

  const loadSignatures = async () => {
    try {
      const keys = await window.storage.list('apartment:');
      if (keys && keys.keys) {
        const sigs = [];
        for (const key of keys.keys) {
          try {
            const result = await window.storage.get(key);
            if (result && result.value) {
              sigs.push(JSON.parse(result.value));
            }
          } catch (e) {
            console.log('Key not found:', key);
          }
        }
        setSignatures(sigs);
      }
    } catch (error) {
      console.error('Error loading signatures:', error);
    }
  };

  const loadPDF = async () => {
    try {
      const result = await window.storage.get('pdf-document');
      if (result && result.value) {
        setPdfData(result.value);
      }
    } catch (error) {
      console.log('No PDF uploaded yet');
    }
  };

  const handlePDFUpload = (e) => {
    const file = e.target.files[0];
    if (!file) return;
    
    if (file.type !== 'application/pdf') {
      alert('❌ אנא העלה קובץ PDF בלבד');
      return;
    }

    if (file.size > 4 * 1024 * 1024) {
      alert('❌ הקובץ גדול מדי. מקסימום 4MB');
      return;
    }

    const reader = new FileReader();
    
    reader.onload = async (event) => {
      try {
        const base64 = event.target.result;
        await window.storage.set('pdf-document', base64);
        setPdfData(base64);
        alert('✅ המסמך הועלה בהצלחה!');
      } catch (error) {
        console.error('Error saving PDF:', error);
        alert('❌ שגיאה בשמירת המסמך. נסה קובץ קטן יותר.');
      }
    };
    
    reader.onerror = () => {
      alert('❌ שגיאה בקריאת הקובץ');
    };
    
    reader.readAsDataURL(file);
  };

  const apartments = Array.from({ length: 32 }, (_, i) => ({
    id: i + 1,
    number: i + 1
  }));

  const startDrawing = (e) => {
    e.preventDefault();
    const canvas = canvasRef.current;
    const rect = canvas.getBoundingClientRect();
    const x = (e.clientX || e.touches?.[0]?.clientX) - rect.left;
    const y = (e.clientY || e.touches?.[0]?.clientY) - rect.top;
    
    const ctx = canvas.getContext('2d');
    ctx.beginPath();
    ctx.moveTo(x, y);
    setIsDrawing(true);
  };

  const draw = (e) => {
    e.preventDefault();
    if (!isDrawing) return;
    
    const canvas = canvasRef.current;
    const rect = canvas.getBoundingClientRect();
    const x = (e.clientX || e.touches?.[0]?.clientX) - rect.left;
    const y = (e.clientY || e.touches?.[0]?.clientY) - rect.top;
    
    const ctx = canvas.getContext('2d');
    ctx.lineTo(x, y);
    ctx.strokeStyle = '#000';
    ctx.lineWidth = 2;
    ctx.stroke();
    setHasSignature(true);
  };

  const stopDrawing = (e) => {
    e.preventDefault();
    setIsDrawing(false);
  };

  const clearSignature = () => {
    const canvas = canvasRef.current;
    const ctx = canvas.getContext('2d');
    ctx.clearRect(0, 0, canvas.width, canvas.height);
    setHasSignature(false);
  };

  const handleSubmitSignature = async () => {
    if (!formData.fullName || !formData.date || !hasSignature) {
      alert('❌ אנא מלא את כל השדות החובה וחתום');
      return;
    }

    const canvas = canvasRef.current;
    const signatureImage = canvas.toDataURL();

    const signatureData = {
      apartmentId: currentApartment,
      fullName: formData.fullName,
      date: formData.date,
      email: formData.email,
      phone: formData.phone,
      signature: signatureImage,
      timestamp: new Date().toISOString()
    };

    try {
      await window.storage.set(`apartment:${currentApartment}`, JSON.stringify(signatureData));
      alert('✅ החתימה נשמרה בהצלחה!');
      setView('dashboard');
      setCurrentApartment(null);
      setFormData({ fullName: '', date: new Date().toISOString().split('T')[0], email: '', phone: '' });
      clearSignature();
      await loadSignatures();
    } catch (error) {
      console.error('Error saving signature:', error);
      alert('❌ אירעה שגיאה בשמירת החתימה');
    }
  };

  const getSignatureStatus = (apartmentId) => {
    return signatures.find(s => s.apartmentId === apartmentId);
  };

  const generateShareLink = (apartmentId) => {
    const currentUrl = window.location.href.split('?')[0];
    return `${currentUrl}?apt=${apartmentId}`;
  };

  const showLinkForApartment = (apartmentId) => {
    setSelectedAptForLink(apartmentId);
    setShowLinkModal(true);
  };

  const copyDirectLink = async (apartmentId) => {
    const link = generateShareLink(apartmentId);
    try {
      await navigator.clipboard.writeText(link);
      alert(`✅ הלינק לדירה ${apartmentId} הועתק!\n\nעכשיו תוכל להדביק אותו בוואטסאפ או במייל.`);
    } catch (error) {
      const textArea = document.createElement('textarea');
      textArea.value = link;
      textArea.style.position = 'fixed';
      textArea.style.left = '-999999px';
      document.body.appendChild(textArea);
      textArea.select();
      try {
        document.execCommand('copy');
        alert(`✅ הלינק הועתק!`);
      } catch (err) {
        alert(`הלינק לדירה ${apartmentId}:\n\n${link}`);
      }
      document.body.removeChild(textArea);
    }
  };

  const testSignature = (apartmentId) => {
    setCurrentApartment(apartmentId);
    setView('sign');
  };

  const downloadAllSignatures = () => {
    const data = JSON.stringify(signatures, null, 2);
    const blob = new Blob([data], { type: 'application/json' });
    const url = URL.createObjectURL(blob);
    const a = document.createElement('a');
    a.href = url;
    a.download = `signatures_oziel_${new Date().toISOString().split('T')[0]}.json`;
    document.body.appendChild(a);
    a.click();
    document.body.removeChild(a);
    URL.revokeObjectURL(url);
  };

  if (view === 'dashboard') {
    const signedCount = signatures.length;
    const pendingCount = 32 - signedCount;
    const percentage = Math.round((signedCount / 32) * 100);

    return (
      <div className="min-h-screen bg-gradient-to-br from-blue-50 to-indigo-100 p-4" dir="rtl">
        {showLinkModal && (
          <div className="fixed inset-0 bg-black bg-opacity-50 flex items-center justify-center z-50 p-4">
            <div className="bg-white rounded-lg shadow-2xl p-6 max-w-2xl w-full">
              <div className="flex justify-between items-center mb-4">
                <h2 className="text-xl font-bold">לינק לחתימה - דירה {selectedAptForLink}</h2>
                <button onClick={() => setShowLinkModal(false)} className="text-gray-500 hover:text-gray-700">
                  <X className="w-6 h-6" />
                </button>
              </div>
              
              <div className="bg-blue-50 border border-blue-300 rounded p-4 mb-4">
                <p className="text-sm text-blue-900 mb-2 font-semibold">📱 העתק את הלינק הזה ושלח בוואטסאפ או במייל:</p>
                <div className="bg-white p-3 rounded border border-blue-200 break-all text-sm font-mono">
                  {generateShareLink(selectedAptForLink)}
                </div>
              </div>

              <button
                onClick={() => {
                  copyDirectLink(selectedAptForLink);
                  setShowLinkModal(false);
                }}
                className="w-full bg-green-600 text-white py-3 rounded-lg hover:bg-green-700 font-semibold flex items-center justify-center gap-2"
              >
                <Copy className="w-5 h-5" />
                העתק לינק ללוח
              </button>
            </div>
          </div>
        )}

        <div className="max-w-6xl mx-auto">
          <div className="bg-white rounded-lg shadow-xl p-6 mb-6">
            <div className="flex items-center justify-between mb-6">
              <div className="flex items-center gap-3">
                <FileText className="w-8 h-8 text-blue-600" />
                <div>
                  <h1 className="text-2xl md:text-3xl font-bold text-gray-800">רחבת הרב עוזיאל 4-14</h1>
                  <p className="text-sm text-gray-600">כניסות זוגיות</p>
                </div>
              </div>
              <div className="flex gap-2">
                <input
                  ref={fileInputRef}
                  type="file"
                  accept="application/pdf"
                  onChange={handlePDFUpload}
                  className="hidden"
                />
                <button
                  onClick={() => fileInputRef.current?.click()}
                  className="flex items-center gap-2 bg-purple-600 text-white px-4 py-2 rounded-lg hover:bg-purple-700 transition-colors"
                >
                  <Upload className="w-5 h-5" />
                  {pdfData ? 'החלף PDF' : 'העלה PDF'}
                </button>
                {signedCount > 0 && (
                  <button
                    onClick={downloadAllSignatures}
                    className="flex items-center gap-2 bg-green-600 text-white px-4 py-2 rounded-lg hover:bg-green-700 transition-colors"
                  >
                    <Download className="w-5 h-5" />
                    הורד חתימות
                  </button>
                )}
              </div>
            </div>

            {!pdfData && (
              <div className="bg-yellow-50 border-2 border-yellow-300 rounded-lg p-4 mb-6">
                <p className="text-yellow-800 font-semibold">⚠️ טרם הועלה מסמך PDF</p>
                <p className="text-yellow-700 text-sm">לחץ על "העלה PDF" להעלאת המסמך (עד 4MB). עד אז יוצג מסמך לדוגמה.</p>
              </div>
            )}

            <div className="grid grid-cols-1 md:grid-cols-4 gap-4 mb-6">
              <div className="bg-green-50 p-4 rounded-lg border border-green-200">
                <div className="flex items-center gap-2 mb-2">
                  <CheckCircle className="w-5 h-5 text-green-600" />
                  <span className="text-sm text-gray-600">חתמו</span>
                </div>
                <div className="text-3xl font-bold text-green-600">{signedCount}</div>
              </div>
              <div className="bg-yellow-50 p-4 rounded-lg border border-yellow-200">
                <div className="flex items-center gap-2 mb-2">
                  <Clock className="w-5 h-5 text-yellow-600" />
                  <span className="text-sm text-gray-600">ממתינים</span>
                </div>
                <div className="text-3xl font-bold text-yellow-600">{pendingCount}</div>
              </div>
              <div className="bg-blue-50 p-4 rounded-lg border border-blue-200">
                <div className="flex items-center gap-2 mb-2">
                  <Users className="w-5 h-5 text-blue-600" />
                  <span className="text-sm text-gray-600">סה"כ דירות</span>
                </div>
                <div className="text-3xl font-bold text-blue-600">32</div>
              </div>
              <div className="bg-purple-50 p-4 rounded-lg border border-purple-200">
                <div className="flex items-center gap-2 mb-2">
                  <FileText className="w-5 h-5 text-purple-600" />
                  <span className="text-sm text-gray-600">אחוז השלמה</span>
                </div>
                <div className="text-3xl font-bold text-purple-600">{percentage}%</div>
              </div>
            </div>

            <div className="mb-4">
              <div className="flex justify-between text-sm text-gray-600 mb-1">
                <span>התקדמות החתימות</span>
                <span>{signedCount} מתוך 32</span>
              </div>
              <div className="w-full bg-gray-200 rounded-full h-3">
                <div 
                  className="bg-gradient-to-r from-blue-500 to-green-500 h-3 rounded-full transition-all duration-500"
                  style={{width: `${percentage}%`}}
                />
              </div>
            </div>
          </div>

          <div className="bg-white rounded-lg shadow-xl p-6">
            <h2 className="text-xl font-bold text-gray-800 mb-4">סטטוס דירות</h2>
            <div className="grid grid-cols-1 md:grid-cols-2 lg:grid-cols-4 gap-4">
              {apartments.map(apt => {
                const signature = getSignatureStatus(apt.id);
                return (
                  <div
                    key={apt.id}
                    className={`p-4 rounded-lg border-2 transition-all ${
                      signature
                        ? 'bg-green-50 border-green-300'
                        : 'bg-gray-50 border-gray-300'
                    }`}
                  >
                    <div className="flex items-center justify-between mb-2">
                      <span className="text-lg font-bold">דירה {apt.number}</span>
                      {signature ? (
                        <CheckCircle className="w-6 h-6 text-green-600" />
                      ) : (
                        <Clock className="w-6 h-6 text-gray-400" />
                      )}
                    </div>
                    {signature ? (
                      <div className="text-sm text-gray-600">
                        <div className="font-semibold">{signature.fullName}</div>
                        <div className="text-xs">{signature.date}</div>
                        {signature.email && <div className="text-xs">{signature.email}</div>}
                        {signature.phone && <div className="text-xs">{signature.phone}</div>}
                      </div>
                    ) : (
                      <div className="space-y-2">
                        <button
                          onClick={() => showLinkForApartment(apt.id)}
                          className="w-full flex items-center justify-center gap-2 bg-blue-600 text-white px-3 py-2 rounded hover:bg-blue-700 text-sm transition-colors"
                        >
                          <Link className="w-4 h-4" />
                          קבל לינק
                        </button>
                        <button
                          onClick={() => testSignature(apt.id)}
                          className="w-full flex items-center justify-center gap-2 bg-gray-600 text-white px-3 py-2 rounded hover:bg-gray-700 text-sm transition-colors"
                        >
                          <Eye className="w-4 h-4" />
                          בדוק
                        </button>
                      </div>
                    )}
                  </div>
                );
              })}
            </div>
          </div>
        </div>
      </div>
    );
  }

  return (
    <div className="min-h-screen bg-gradient-to-br from-blue-50 to-indigo-100 p-4" dir="rtl">
      <div className="max-w-5xl mx-auto">
        <div className="bg-white rounded-lg shadow-xl p-6">
          <div className="flex items-center justify-between mb-6">
            <div>
              <h1 className="text-xl md:text-2xl font-bold text-gray-800">רחבת הרב עוזיאל 4-14 - כניסות זוגיות</h1>
              <p className="text-sm text-gray-600">חתימה על מסמך - דירה {currentApartment}</p>
            </div>
            <button
              onClick={() => {
                setView('dashboard');
                setCurrentApartment(null);
              }}
              className="text-gray-600 hover:text-gray-800"
            >
              <X className="w-6 h-6" />
            </button>
          </div>

          {pdfData ? (
            <div className="mb-6 border-2 border-gray-300 rounded-lg overflow-hidden">
              <iframe
                src={pdfData}
                className="w-full h-96"
                title="PDF Document"
              />
            </div>
          ) : (
            <div className="mb-6 border-2 border-gray-300 rounded-lg p-6 bg-gray-50 max-h-96 overflow-y-auto">
              <div className="text-center mb-4">
                <h2 className="text-xl font-bold text-gray-800">מסמך הסכם בעלי דירות</h2>
                <p className="text-sm text-gray-600">רחבת הרב עוזיאל 4-14 | כניסות זוגיות</p>
              </div>
              
              <div className="bg-white p-6 rounded shadow-sm mb-4">
                <h3 className="font-bold text-lg mb-3">הסכם בעלי דירות - בית משותף</h3>
                <p className="text-sm leading-relaxed mb-3">
                  אנו החתומים מטה, בעלי הדירות ברחבת הרב עוזיאל 4-14, מצהירים ומתחייבים בזאת כדלקמן:
                </p>
                <ol className="text-sm space-y-2 mr-5">
                  <li>1. אנו מסכימים לכל התנאים והסידורים המפורטים במסמך זה.</li>
                  <li>2. אנו מתחייבים לשלם את חלקנו היחסי בהוצאות המשותפות של הבניין.</li>
                  <li>3. אנו מסכימים לקיים את תקנון הבית המשותף ולפעול על פיו.</li>
                  <li>4. אנו מאשרים את ביצוע העבודות המתוכננות בבניין כמפורט בנספח א'.</li>
                </ol>
              </div>

              <div className="bg-white p-6 rounded shadow-sm">
                <h3 className="font-bold mb-3">טבלת חתימות בעלי הדירות</h3>
                <div className="overflow-x-auto">
                  <table className="w-full border-collapse border border-gray-300 text-sm">
                    <thead>
                      <tr className="bg-gray-100">
                        <th className="border border-gray-300 p-2">מס' דירה</th>
                        <th className="border border-gray-300 p-2">שם מלא</th>
                        <th className="border border-gray-300 p-2">תאריך</th>
                        <th className="border border-gray-300 p-2">חתימה</th>
                      </tr>
                    </thead>
                    <tbody>
                      {apartments.slice(0, 10).map(apt => {
                        const sig = getSignatureStatus(apt.id);
                        return (
                          <tr key={apt.id} className={apt.id === currentApartment ? 'bg-yellow-50' : ''}>
                            <td className="border border-gray-300 p-2 text-center">{apt.number}</td>
                            <td className="border border-gray-300 p-2">{sig?.fullName || '_______________'}</td>
                            <td className="border border-gray-300 p-2">{sig?.date || '_______________'}</td>
                            <td className="border border-gray-300 p-2 text-center">
                              {sig ? '✓' : '_______________'}
                            </td>
                          </tr>
                        );
                      })}
                    </tbody>
                  </table>
                </div>
                <p className="text-xs text-gray-500 mt-2">* הטבלה מציגה 10 דירות ראשונות לדוגמה</p>
              </div>
            </div>
          )}

          <div className="bg-blue-50 p-4 rounded-lg mb-6 border border-blue-200">
            <h3 className="font-bold text-blue-900 mb-2">חתימה על המסמך - דירה {currentApartment}</h3>
            <p className="text-sm text-blue-800">
              אנא מלא את הפרטים הבאים וחתום במקום המיועד להשלמת תהליך החתימה על המסמך.
            </p>
          </div>

          <div className="space-y-4">
            <div>
              <label className="block text-sm font-medium text-gray-700 mb-1">
                שם מלא <span className="text-red-600">*</span>
              </label>
              <input
                type="text"
                value={formData.fullName}
                onChange={(e) => setFormData({...formData, fullName: e.target.value})}
                className="w-full px-4 py-2 border border-gray-300 rounded-lg focus:ring-2 focus:ring-blue-500 focus:border-transparent"
                placeholder="הזן שם מלא"
              />
            </div>

            <div>
              <label className="block text-sm font-medium text-gray-700 mb-1">
                תאריך <span className="text-red-600">*</span>
              </label>
              <input
                type="date"
                value={formData.date}
                onChange={(e) => setFormData({...formData, date: e.target.value})}
                className="w-full px-4 py-2 border border-gray-300 rounded-lg focus:ring-2 focus:ring-blue-500 focus:border-transparent"
              />
            </div>

            <div>
              <label className="block text-sm font-medium text-gray-700 mb-1">
                מייל
              </label>
              <input
                type="email"
                value={formData.email}
                onChange={(e) => setFormData({...formData, email: e.target.value})}
                className="w-full px-4 py-2 border border-gray-300 rounded-lg focus:ring-2 focus:ring-blue-500 focus:border-transparent"
                placeholder="example@email.com"
              />
            </div>

            <div>
              <label className="block text-sm font-medium text-gray-700 mb-1">
                טלפון
              </label>
              <input
                type="tel"
                value={formData.phone}
                onChange={(e) => setFormData({...formData, phone: e.target.value})}
                className="w-full px-4 py-2 border border-gray-300 rounded-lg focus:ring-2 focus:ring-blue-500 focus:border-transparent"
                placeholder="050-1234567"
              />
            </div>

            <div>
              <label className="block text-sm font-medium text-gray-700 mb-2">
                חתימה דיגיטלית <span className="text-red-600">*</span>
              </label>
              <p className="text-xs text-gray-500 mb-2">צייר את חתימתך במסך באמצעות העכבר או האצבע</p>
              <div className="border-2 border-gray-300 rounded-lg overflow-hidden bg-white">
                <canvas
                  ref={canvasRef}
                  width={700}
                  height={200}
                  className="w-full cursor-crosshair touch-none"
                  onMouseDown={startDrawing}
                  onMouseMove={draw}
                  onMouseUp={stopDrawing}
                  onMouseLeave={stopDrawing}
                  onTouchStart={startDrawing}
                  onTouchMove={draw}
                  onTouchEnd={stopDrawing}
                />
              </div>
              <button
                onClick={clearSignature}
                className="mt-2 text-sm text-red-600 hover:text-red-800 font-medium"
              >
                נקה חתימה
              </button>
            </div>

            <button
              onClick={handleSubmitSignature}
              className="w-full bg-blue-600 text-white py-3 rounded-lg hover:bg-blue-700 font-semibold text-lg transition-colors"
            >
              אישור וחתימה על המסמך
            </button>
          </div>
        </div>
      </div>
    </div>
  );
};

export default ApartmentSignatureApp;
