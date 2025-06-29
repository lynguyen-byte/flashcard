import React, { useState, useEffect, useRef, useCallback } from 'react';
import { initializeApp } from 'firebase/app';
import { getAuth, signInAnonymously, signInWithCustomToken, onAuthStateChanged } from 'firebase/auth';
import { getFirestore, collection, addDoc, getDocs, doc, setDoc, deleteDoc, onSnapshot, query } from 'firebase/firestore';

// Define Firebase config and app ID from global variables
const appId = typeof __app_id !== 'undefined' ? __app_id : 'default-app-id';
const firebaseConfig = typeof __firebase_config !== 'undefined' ? JSON.parse(__firebase_config) : {};
const initialAuthToken = typeof __initial_auth_token !== 'undefined' ? __initial_auth_token : null;

// Main App Component
const App = () => {
    const [db, setDb] = useState(null);
    const [auth, setAuth] = useState(null);
    const [userId, setUserId] = useState(null);
    const [loading, setLoading] = useState(true);
    const [currentView, setCurrentView] = useState('manage'); // 'manage', 'review', 'test', 'flipcard'
    const [message, setMessage] = useState('');
    const [messageType, setMessageType] = useState(''); // 'success', 'error', 'info'

    // Initialize Firebase and authenticate user
    useEffect(() => {
        const initFirebase = async () => {
            try {
                const app = initializeApp(firebaseConfig);
                const authInstance = getAuth(app);
                const dbInstance = getFirestore(app);

                setAuth(authInstance);
                setDb(dbInstance);

                // Sign in with custom token or anonymously
                if (initialAuthToken) {
                    await signInWithCustomToken(authInstance, initialAuthToken);
                } else {
                    await signInAnonymously(authInstance);
                }

                // Listen for auth state changes
                const unsubscribe = onAuthStateChanged(authInstance, (user) => {
                    if (user) {
                        setUserId(user.uid);
                    } else {
                        // Fallback for anonymous user ID if somehow auth fails
                        setUserId(crypto.randomUUID());
                    }
                    setLoading(false);
                });

                return () => unsubscribe();
            } catch (error) {
                console.error("Lỗi khởi tạo Firebase hoặc xác thực:", error);
                setMessage('Lỗi khi khởi tạo ứng dụng. Vui lòng thử lại.');
                setMessageType('error');
                setLoading(false);
            }
        };

        initFirebase();
    }, []);

    // Function to show transient messages
    const showMessage = useCallback((msg, type) => {
        setMessage(msg);
        setMessageType(type);
        const timer = setTimeout(() => {
            setMessage('');
            setMessageType('');
        }, 3000);
        return () => clearTimeout(timer);
    }, []);

    if (loading) {
        return (
            <div className="flex items-center justify-center min-h-screen bg-gray-100">
                <div className="text-xl text-gray-700">Đang tải ứng dụng...</div>
            </div>
        );
    }

    return (
        <div className="min-h-screen bg-gray-100 font-inter p-4 flex flex-col items-center">
            <h1 className="text-4xl font-bold text-blue-800 mb-6 text-center">
                Ứng Dụng Flashcard
            </h1>

            {userId && (
                <div className="text-sm text-gray-600 mb-4">
                    ID người dùng của bạn: <span className="font-mono bg-gray-200 px-2 py-1 rounded">{userId}</span>
                </div>
            )}

            {message && (
                <div
                    className={`p-3 mb-4 rounded-lg shadow-md text-sm ${
                        messageType === 'success' ? 'bg-green-100 text-green-700' :
                        messageType === 'error' ? 'bg-red-100 text-red-700' :
                        'bg-blue-100 text-blue-700'
                    }`}
                >
                    {message}
                </div>
            )}

            <nav className="mb-8 flex space-x-4">
                <button
                    onClick={() => setCurrentView('manage')}
                    className={`px-6 py-3 rounded-full font-semibold transition-all duration-200 ${
                        currentView === 'manage' ? 'bg-blue-600 text-white shadow-lg' : 'bg-white text-blue-600 border border-blue-300 hover:bg-blue-50'
                    }`}
                >
                    Quản lý Từ vựng
                </button>
                <button
                    onClick={() => setCurrentView('review')}
                    className={`px-6 py-3 rounded-full font-semibold transition-all duration-200 ${
                        currentView === 'review' || currentView === 'test' || currentView === 'flipcard' ? 'bg-blue-600 text-white shadow-lg' : 'bg-white text-blue-600 border border-blue-300 hover:bg-blue-50'
                    }`}
                >
                    Ôn tập / Kiểm tra
                </button>
            </nav>

            <div className="w-full max-w-4xl bg-white p-8 rounded-xl shadow-2xl">
                {currentView === 'manage' && db && auth && userId && (
                    <ManageVocabulary db={db} auth={auth} userId={userId} showMessage={showMessage} />
                )}
                {(currentView === 'review' || currentView === 'test' || currentView === 'flipcard') && db && auth && userId && (
                    <ReviewSection db={db} auth={auth} userId={userId} showMessage={showMessage} setCurrentView={setCurrentView} currentView={currentView} />
                )}
            </div>
        </div>
    );
};

// Component to manage vocabulary (add, import, list)
const ManageVocabulary = ({ db, auth, userId, showMessage }) => {
    const [lessons, setLessons] = useState([]);
    const [flashcards, setFlashcards] = useState([]);
    const [newLessonName, setNewLessonName] = useState('');
    const [englishWord, setEnglishWord] = useState('');
    const [vietnameseWord, setVietnameseWord] = useState('');
    const [selectedLessonId, setSelectedLessonId] = useState('');
    const [bulkText, setBulkText] = useState('');
    const [bulkLessonId, setBulkLessonId] = useState('');
    const [imageFile, setImageFile] = useState(null);
    const [ocrLoading, setOcrLoading] = useState(false);

    // Fetch lessons and flashcards on component mount and on data changes
    useEffect(() => {
        if (!db || !userId) return;

        // Listener for lessons
        const lessonsQuery = query(collection(db, `artifacts/${appId}/users/${userId}/lessons`));
        const unsubscribeLessons = onSnapshot(lessonsQuery, (snapshot) => {
            const lessonsData = snapshot.docs.map(doc => ({ id: doc.id, ...doc.data() }));
            setLessons(lessonsData);
            if (lessonsData.length > 0 && !selectedLessonId) {
                // Set default selected lesson if not already set and lessons exist
                setSelectedLessonId(lessonsData[0].id);
                setBulkLessonId(lessonsData[0].id);
            }
        }, (error) => {
            console.error("Error loading lessons:", error);
            showMessage('Không thể tải bài học.', 'error');
        });

        // Listener for flashcards
        const flashcardsQuery = query(collection(db, `artifacts/${appId}/users/${userId}/flashcards`));
        const unsubscribeFlashcards = onSnapshot(flashcardsQuery, (snapshot) => {
            const flashcardsData = snapshot.docs.map(doc => ({ id: doc.id, ...doc.data() }));
            setFlashcards(flashcardsData);
        }, (error) => {
            console.error("Error loading flashcards:", error);
            showMessage('Không thể tải flashcard.', 'error');
        });

        return () => {
            unsubscribeLessons();
            unsubscribeFlashcards();
        };
    }, [db, userId, showMessage, selectedLessonId]);

    // Handle adding a new lesson
    const handleAddLesson = async () => {
        if (!newLessonName.trim() || !db || !userId) {
            showMessage('Tên bài học không được trống.', 'error');
            return;
        }
        try {
            const docRef = await addDoc(collection(db, `artifacts/${appId}/users/${userId}/lessons`), {
                name: newLessonName.trim(),
                createdAt: new Date(),
            });
            setNewLessonName('');
            showMessage('Thêm bài học thành công!', 'success');
        } catch (e) {
            console.error("Error adding lesson:", e);
            showMessage('Lỗi khi thêm bài học.', 'error');
        }
    };

    // Handle adding a single flashcard
    const handleAddFlashcard = async () => {
        if (!englishWord.trim() || !vietnameseWord.trim() || !selectedLessonId || !db || !userId) {
            showMessage('Vui lòng điền đầy đủ từ vựng và chọn bài học.', 'error');
            return;
        }
        try {
            await addDoc(collection(db, `artifacts/${appId}/users/${userId}/flashcards`), {
                englishWord: englishWord.trim(),
                vietnameseWord: vietnameseWord.trim(),
                lessonId: selectedLessonId,
                createdAt: new Date(),
            });
            setEnglishWord('');
            setVietnameseWord('');
            showMessage('Thêm flashcard thành công!', 'success');
        } catch (e) {
            console.error("Error adding flashcard:", e);
            showMessage('Lỗi khi thêm flashcard.', 'error');
        }
    };

    // Handle bulk import of flashcards
    const handleBulkImport = async () => {
        if (!bulkText.trim() || !bulkLessonId || !db || !userId) {
            showMessage('Vui lòng nhập văn bản và chọn bài học.', 'error');
            return;
        }

        const lines = bulkText.split('\n').filter(line => line.trim() !== '');
        const newFlashcards = [];
        for (const line of lines) {
            const parts = line.split('-').map(part => part.trim());
            if (parts.length >= 2) {
                newFlashcards.push({
                    englishWord: parts[0],
                    vietnameseWord: parts.slice(1).join(' - '), // Join back if meaning has hyphens
                    lessonId: bulkLessonId,
                    createdAt: new Date(),
                });
            }
        }

        if (newFlashcards.length === 0) {
            showMessage('Không tìm thấy từ vựng hợp lệ nào trong văn bản.', 'error');
            return;
        }

        try {
            await Promise.all(newFlashcards.map(card =>
                addDoc(collection(db, `artifacts/${appId}/users/${userId}/flashcards`), card)
            ));
            setBulkText('');
            showMessage(`Đã thêm ${newFlashcards.length} flashcard thành công!`, 'success');
        } catch (e) {
            console.error("Error bulk importing:", e);
            showMessage('Lỗi khi nhập hàng loạt flashcard.', 'error');
        }
    };

    // Handle image upload and OCR
    const handleImageUploadAndOCR = async (e) => {
        const file = e.target.files[0];
        if (!file) return;

        setImageFile(file);
        setOcrLoading(true);
        showMessage('Đang phân tích hình ảnh, vui lòng đợi...', 'info');

        const reader = new FileReader();
        reader.onloadend = async () => {
            const base64ImageData = reader.result.split(',')[1]; // Get base64 string without data:image/png;base64,

            try {
                let chatHistory = [];
                const prompt = "Extract all English words and their Vietnamese meanings from this image. Present them as 'English Word - Vietnamese Meaning' on separate lines. If no Vietnamese meaning is available, just provide the English word. Ignore any other text not related to vocabulary pairs.";
                chatHistory.push({ role: "user", parts: [{ text: prompt }] });

                const payload = {
                    contents: [
                        {
                            role: "user",
                            parts: [
                                { text: prompt },
                                {
                                    inlineData: {
                                        mimeType: file.type,
                                        data: base64ImageData
                                    }
                                }
                            ]
                        }
                    ],
                };
                const apiKey = "";
                const apiUrl = `https://generativelanguage.googleapis.com/v1beta/models/gemini-2.0-flash:generateContent?key=${apiKey}`;

                const response = await fetch(apiUrl, {
                    method: 'POST',
                    headers: { 'Content-Type': 'application/json' },
                    body: JSON.stringify(payload)
                });

                const result = await response.json();
                if (result.candidates && result.candidates.length > 0 &&
                    result.candidates[0].content && result.candidates[0].content.parts &&
                    result.candidates[0].content.parts.length > 0) {
                    const extractedText = result.candidates[0].content.parts[0].text;
                    setBulkText(extractedText);
                    showMessage('Đã trích xuất văn bản thành công! Vui lòng kiểm tra và thêm vào bài học.', 'success');
                } else {
                    showMessage('Không thể trích xuất văn bản từ hình ảnh. Vui lòng thử lại.', 'error');
                    console.error("Unexpected API response structure:", result);
                }
            } catch (error) {
                console.error("Error during OCR process:", error);
                showMessage('Lỗi khi xử lý hình ảnh: ' + error.message, 'error');
            } finally {
                setOcrLoading(false);
            }
        };
        reader.readAsDataURL(file);
    };

    // Delete lesson and associated flashcards
    const handleDeleteLesson = async (lessonId) => {
        if (!db || !userId) return;

        // Custom confirmation modal (instead of alert/confirm)
        const userConfirmed = await new Promise(resolve => {
            const confirmModal = document.createElement('div');
            confirmModal.className = "fixed inset-0 bg-gray-600 bg-opacity-50 flex items-center justify-center z-50";
            confirmModal.innerHTML = `
                <div class="bg-white p-8 rounded-lg shadow-xl max-w-sm w-full">
                    <h3 class="text-lg font-semibold mb-4">Xác nhận xóa</h3>
                    <p class="mb-6">Bạn có chắc chắn muốn xóa bài học này và tất cả từ vựng liên quan không?</p>
                    <div class="flex justify-end space-x-3">
                        <button id="cancelBtn" class="px-4 py-2 bg-gray-200 text-gray-800 rounded-md hover:bg-gray-300">Hủy</button>
                        <button id="confirmBtn" class="px-4 py-2 bg-red-600 text-white rounded-md hover:bg-red-700">Xóa</button>
                    </div>
                </div>
            `;
            document.body.appendChild(confirmModal);

            document.getElementById('confirmBtn').onclick = () => {
                document.body.removeChild(confirmModal);
                resolve(true);
            };
            document.getElementById('cancelBtn').onclick = () => {
                document.body.removeChild(confirmModal);
                resolve(false);
            };
        });

        if (!userConfirmed) return;

        try {
            // Delete lesson document
            await deleteDoc(doc(db, `artifacts/${appId}/users/${userId}/lessons`, lessonId));

            // Delete associated flashcards
            const cardsToDelete = flashcards.filter(card => card.lessonId === lessonId);
            await Promise.all(cardsToDelete.map(card =>
                deleteDoc(doc(db, `artifacts/${appId}/users/${userId}/flashcards`, card.id))
            ));
            showMessage('Xóa bài học và các flashcard liên quan thành công!', 'success');
        } catch (error) {
            console.error("Error deleting lesson:", error);
            showMessage('Lỗi khi xóa bài học.', 'error');
        }
    };

    // Delete a single flashcard
    const handleDeleteFlashcard = async (cardId) => {
        if (!db || !userId) return;

        const userConfirmed = await new Promise(resolve => {
            const confirmModal = document.createElement('div');
            confirmModal.className = "fixed inset-0 bg-gray-600 bg-opacity-50 flex items-center justify-center z-50";
            confirmModal.innerHTML = `
                <div class="bg-white p-8 rounded-lg shadow-xl max-w-sm w-full">
                    <h3 class="text-lg font-semibold mb-4">Xác nhận xóa</h3>
                    <p class="mb-6">Bạn có chắc chắn muốn xóa flashcard này?</p>
                    <div class="flex justify-end space-x-3">
                        <button id="cancelBtn" class="px-4 py-2 bg-gray-200 text-gray-800 rounded-md hover:bg-gray-300">Hủy</button>
                        <button id="confirmBtn" class="px-4 py-2 bg-red-600 text-white rounded-md hover:bg-red-700">Xóa</button>
                    </div>
                </div>
            `;
            document.body.appendChild(confirmModal);

            document.getElementById('confirmBtn').onclick = () => {
                document.body.removeChild(confirmModal);
                resolve(true);
            };
            document.getElementById('cancelBtn').onclick = () => {
                document.body.removeChild(confirmModal);
                resolve(false);
            };
        });

        if (!userConfirmed) return;

        try {
            await deleteDoc(doc(db, `artifacts/${appId}/users/${userId}/flashcards`, cardId));
            showMessage('Xóa flashcard thành công!', 'success');
        } catch (error) {
            console.error("Error deleting flashcard:", error);
            showMessage('Lỗi khi xóa flashcard.', 'error');
        }
    };


    return (
        <div className="space-y-8">
            {/* Thêm bài học mới */}
            <div className="bg-blue-50 p-6 rounded-lg shadow-inner">
                <h2 className="text-2xl font-semibold text-blue-700 mb-4">Thêm Bài học mới</h2>
                <div className="flex flex-col sm:flex-row gap-3">
                    <input
                        type="text"
                        placeholder="Tên bài học (VD: Bài 1, Từ vựng IELTS...)"
                        value={newLessonName}
                        onChange={(e) => setNewLessonName(e.target.value)}
                        className="flex-grow p-3 border border-blue-300 rounded-md focus:ring-2 focus:ring-blue-400 outline-none"
                    />
                    <button
                        onClick={handleAddLesson}
                        className="px-6 py-3 bg-blue-600 text-white rounded-md font-semibold hover:bg-blue-700 transition-colors shadow-md"
                    >
                        Thêm Bài học
                    </button>
                </div>
            </div>

            {/* Nhập từ vựng lẻ */}
            <div className="bg-green-50 p-6 rounded-lg shadow-inner">
                <h2 className="text-2xl font-semibold text-green-700 mb-4">Nhập Từ vựng lẻ</h2>
                <div className="grid grid-cols-1 md:grid-cols-2 gap-4">
                    <input
                        type="text"
                        placeholder="Từ tiếng Anh"
                        value={englishWord}
                        onChange={(e) => setEnglishWord(e.target.value)}
                        className="p-3 border border-green-300 rounded-md focus:ring-2 focus:ring-green-400 outline-none"
                    />
                    <input
                        type="text"
                        placeholder="Nghĩa tiếng Việt"
                        value={vietnameseWord}
                        onChange={(e) => setVietnameseWord(e.target.value)}
                        className="p-3 border border-green-300 rounded-md focus:ring-2 focus:ring-green-400 outline-none"
                    />
                    <select
                        value={selectedLessonId}
                        onChange={(e) => setSelectedLessonId(e.target.value)}
                        className="p-3 border border-green-300 rounded-md focus:ring-2 focus:ring-green-400 outline-none col-span-1 md:col-span-2"
                    >
                        <option value="">-- Chọn bài học --</option>
                        {lessons.map(lesson => (
                            <option key={lesson.id} value={lesson.id}>{lesson.name}</option>
                        ))}
                    </select>
                </div>
                <button
                    onClick={handleAddFlashcard}
                    className="mt-4 w-full px-6 py-3 bg-green-600 text-white rounded-md font-semibold hover:bg-green-700 transition-colors shadow-md"
                >
                    Thêm Flashcard
                </button>
            </div>

            {/* Nhập từ vựng hàng loạt từ văn bản hoặc hình ảnh */}
            <div className="bg-yellow-50 p-6 rounded-lg shadow-inner">
                <h2 className="text-2xl font-semibold text-yellow-700 mb-4">Nhập Từ vựng hàng loạt</h2>
                <p className="text-sm text-gray-600 mb-3">
                    Mỗi dòng một từ, định dạng: <span className="font-mono bg-yellow-100 p-1 rounded">English Word - Nghĩa tiếng Việt</span>
                </p>
                <textarea
                    placeholder="Nhập từ vựng hàng loạt vào đây (VD: Hello - Xin chào&#10;Goodbye - Tạm biệt) hoặc sử dụng tính năng 'Nhận diện từ hình ảnh' bên dưới."
                    value={bulkText}
                    onChange={(e) => setBulkText(e.target.value)}
                    rows="6"
                    className="w-full p-3 border border-yellow-300 rounded-md focus:ring-2 focus:ring-yellow-400 outline-none resize-y"
                ></textarea>

                <div className="mt-4 mb-4">
                    <label className="block text-gray-700 text-sm font-bold mb-2">
                        Nhận diện từ vựng từ hình ảnh:
                    </label>
                    <input
                        type="file"
                        accept="image/*"
                        onChange={handleImageUploadAndOCR}
                        className="w-full text-gray-700 bg-white border border-yellow-300 rounded-md p-2 cursor-pointer"
                        disabled={ocrLoading}
                    />
                    {ocrLoading && (
                        <p className="text-blue-500 text-sm mt-2">Đang xử lý hình ảnh... Vui lòng đợi.</p>
                    )}
                </div>

                <select
                    value={bulkLessonId}
                    onChange={(e) => setBulkLessonId(e.target.value)}
                    className="mt-3 w-full p-3 border border-yellow-300 rounded-md focus:ring-2 focus:ring-yellow-400 outline-none"
                >
                    <option value="">-- Chọn bài học cho hàng loạt --</option>
                    {lessons.map(lesson => (
                        <option key={lesson.id} value={lesson.id}>{lesson.name}</option>
                    ))}
                </select>
                <button
                    onClick={handleBulkImport}
                    className="mt-4 w-full px-6 py-3 bg-yellow-600 text-white rounded-md font-semibold hover:bg-yellow-700 transition-colors shadow-md"
                >
                    Thêm Từ vựng hàng loạt
                </button>
            </div>

            {/* Danh sách Bài học và Từ vựng */}
            <div className="bg-gray-50 p-6 rounded-lg shadow-inner">
                <h2 className="text-2xl font-semibold text-gray-700 mb-4">Danh sách Bài học và Từ vựng</h2>
                {lessons.length === 0 ? (
                    <p className="text-gray-500">Chưa có bài học nào. Hãy thêm một bài học mới!</p>
                ) : (
                    <div className="space-y-6">
                        {lessons.map(lesson => (
                            <div key={lesson.id} className="border border-gray-200 rounded-lg p-4 bg-white shadow-sm">
                                <div className="flex justify-between items-center mb-3">
                                    <h3 className="text-xl font-medium text-gray-800">
                                        {lesson.name} ({flashcards.filter(card => card.lessonId === lesson.id).length} từ)
                                    </h3>
                                    <button
                                        onClick={() => handleDeleteLesson(lesson.id)}
                                        className="text-red-500 hover:text-red-700 transition-colors"
                                        title="Xóa bài học và tất cả từ vựng"
                                    >
                                        <svg xmlns="http://www.w3.org/2000/svg" className="h-6 w-6" fill="none" viewBox="0 0 24 24" stroke="currentColor">
                                            <path strokeLinecap="round" strokeLinejoin="round" strokeWidth={2} d="M19 7l-.867 12.142A2 2 0 0116.138 21H7.862a2 2 0 01-1.995-1.858L5 7m5 4v6m4-6v6m1-10V4a1 1 0 00-1-1h-4a1 1 0 00-1 1v3M4 7h16" />
                                        </svg>
                                    </button>
                                </div>
                                {flashcards.filter(card => card.lessonId === lesson.id).length > 0 ? (
                                    <ul className="divide-y divide-gray-100">
                                        {flashcards
                                            .filter(card => card.lessonId === lesson.id)
                                            .map(card => (
                                                <li key={card.id} className="flex justify-between items-center py-2">
                                                    <span className="text-gray-700">
                                                        <span className="font-semibold">{card.englishWord}</span> - {card.vietnameseWord}
                                                    </span>
                                                    <button
                                                        onClick={() => handleDeleteFlashcard(card.id)}
                                                        className="text-red-400 hover:text-red-600 text-sm"
                                                        title="Xóa flashcard"
                                                    >
                                                        Xóa
                                                    </button>
                                                </li>
                                            ))}
                                    </ul>
                                ) : (
                                    <p className="text-gray-500 text-sm">Chưa có từ vựng nào trong bài học này.</p>
                                )}
                            </div>
                        ))}
                    </div>
                )}
            </div>
        </div>
    );
};

// Component for Review Configuration and starting tests
const ReviewSection = ({ db, auth, userId, showMessage, setCurrentView, currentView }) => {
    const [lessons, setLessons] = useState([]);
    const [flashcards, setFlashcards] = useState([]);
    const [selectedLessonIds, setSelectedLessonIds] = useState([]);
    const [numQuestions, setNumQuestions] = useState(10);
    const [testDuration, setTestDuration] = useState(60); // in seconds, for test mode
    const [testMode, setTestMode] = useState('flip-card'); // 'eng-vie', 'vie-eng', 'flip-card'
    const [testFlashcards, setTestFlashcards] = useState([]); // Cards selected for the current test/flip session
    const [lastTestFlashcards, setLastTestFlashcards] = useState([]); // Stores cards from the last full test for re-testing

    // Fetch lessons and flashcards for review configuration
    useEffect(() => {
        if (!db || !userId) return;

        const lessonsQuery = query(collection(db, `artifacts/${appId}/users/${userId}/lessons`));
        const unsubscribeLessons = onSnapshot(lessonsQuery, (snapshot) => {
            const lessonsData = snapshot.docs.map(doc => ({ id: doc.id, ...doc.data() }));
            setLessons(lessonsData);
        }, (error) => {
            console.error("Error loading lessons for review:", error);
            showMessage('Không thể tải bài học cho phần ôn tập.', 'error');
        });

        const flashcardsQuery = query(collection(db, `artifacts/${appId}/users/${userId}/flashcards`));
        const unsubscribeFlashcards = onSnapshot(flashcardsQuery, (snapshot) => {
            const flashcardsData = snapshot.docs.map(doc => ({ id: doc.id, ...doc.data() }));
            setFlashcards(flashcardsData);
        }, (error) => {
            console.error("Error loading flashcards for review:", error);
            showMessage('Không thể tải flashcard cho phần ôn tập.', 'error');
        });

        return () => {
            unsubscribeLessons();
            unsubscribeFlashcards();
        };
    }, [db, userId, showMessage]);

    // Handle lesson selection (multi-select)
    const handleLessonSelection = (e) => {
        const { value, checked } = e.target;
        if (checked) {
            setSelectedLessonIds(prev => [...prev, value]);
        } else {
            setSelectedLessonIds(prev => prev.filter(id => id !== value));
        }
    };

    // Prepare and start the test/flip card session
    const startReviewSession = (useLastTest = false) => {
        let cardsToUse = [];

        if (useLastTest && lastTestFlashcards.length > 0) {
            // Use the exact same content, but reshuffle the order
            cardsToUse = [...lastTestFlashcards].sort(() => 0.5 - Math.random()); // Reshuffle
            showMessage('Đang làm lại bài kiểm tra cũ với thứ tự mới.', 'info');
        } else {
            if (flashcards.length === 0) {
                showMessage('Chưa có từ vựng nào để ôn tập.', 'error');
                return;
            }

            let filteredCards = flashcards;
            if (selectedLessonIds.length > 0) {
                filteredCards = flashcards.filter(card => selectedLessonIds.includes(card.lessonId));
            }

            if (filteredCards.length === 0) {
                showMessage('Không có từ vựng nào trong các bài học đã chọn.', 'error');
                return;
            }

            // Shuffle cards and select number of questions
            const shuffledCards = filteredCards.sort(() => 0.5 - Math.random());
            cardsToUse = shuffledCards.slice(0, Math.min(numQuestions, shuffledCards.length));

            if (cardsToUse.length === 0) {
                showMessage('Không đủ từ vựng để tạo bài ôn tập/kiểm tra với số lượng yêu cầu.', 'error');
                return;
            }
            // Save this new set for potential re-testing
            setLastTestFlashcards(cardsToUse);
            showMessage('Bắt đầu bài kiểm tra mới.', 'success');
        }

        setTestFlashcards(cardsToUse);
        if (testMode === 'flip-card') {
            setCurrentView('flipcard');
        } else {
            setCurrentView('test');
        }
    };

    // Render the appropriate component based on currentView
    if (currentView === 'test') {
        return (
            <Test
                flashcards={testFlashcards}
                duration={testDuration}
                mode={testMode}
                onTestEnd={() => {
                    setCurrentView('review'); // Go back to review config after test
                }}
            />
        );
    } else if (currentView === 'flipcard') {
        return (
            <FlipCardMode
                flashcards={testFlashcards}
                onEnd={() => {
                    setCurrentView('review'); // Go back to review config after flip card session
                }}
            />
        );
    }

    // Default: Render review configuration
    return (
        <div className="p-6 rounded-lg shadow-inner bg-purple-50">
            <h2 className="text-2xl font-semibold text-purple-700 mb-4">Cấu hình Ôn tập</h2>

            <div className="mb-6">
                <label className="block text-gray-700 text-sm font-bold mb-2">
                    Chọn Bài học (có thể chọn nhiều để ôn tập hỗn hợp):
                </label>
                <div className="grid grid-cols-1 sm:grid-cols-2 md:grid-cols-3 gap-2 max-h-48 overflow-y-auto border border-purple-200 rounded-md p-3 bg-white">
                    {lessons.length === 0 ? (
                        <p className="text-gray-500 col-span-full">Chưa có bài học nào để chọn.</p>
                    ) : (
                        lessons.map(lesson => (
                            <label key={lesson.id} className="inline-flex items-center text-gray-700">
                                <input
                                    type="checkbox"
                                    value={lesson.id}
                                    checked={selectedLessonIds.includes(lesson.id)}
                                    onChange={handleLessonSelection}
                                    className="form-checkbox h-5 w-5 text-purple-600 rounded"
                                />
                                <span className="ml-2">{lesson.name}</span>
                            </label>
                        ))
                    )}
                </div>
            </div>

            <div className="mb-6">
                <label className="block text-gray-700 text-sm font-bold mb-2" htmlFor="numQuestions">
                    Số lượng câu hỏi / thẻ muốn ôn tập:
                </label>
                <input
                    type="number"
                    id="numQuestions"
                    min="1"
                    value={numQuestions}
                    onChange={(e) => setNumQuestions(Math.max(1, parseInt(e.target.value) || 1))}
                    className="w-full p-3 border border-purple-300 rounded-md focus:ring-2 focus:ring-purple-400 outline-none"
                />
            </div>

            {testMode !== 'flip-card' && ( // Only show duration for test mode
                <div className="mb-6">
                    <label className="block text-gray-700 text-sm font-bold mb-2" htmlFor="testDuration">
                        Thời gian làm bài (giây):
                    </label>
                    <input
                        type="number"
                        id="testDuration"
                        min="10"
                        value={testDuration}
                        onChange={(e) => setTestDuration(Math.max(10, parseInt(e.target.value) || 10))}
                        className="w-full p-3 border border-purple-300 rounded-md focus:ring-2 focus:ring-purple-400 outline-none"
                    />
                </div>
            )}

            <div className="mb-6">
                <label className="block text-gray-700 text-sm font-bold mb-2">
                    Chế độ ôn tập:
                </label>
                <div className="flex flex-col sm:flex-row space-y-2 sm:space-y-0 sm:space-x-4">
                    <label className="inline-flex items-center">
                        <input
                            type="radio"
                            name="testMode"
                            value="flip-card"
                            checked={testMode === 'flip-card'}
                            onChange={(e) => setTestMode(e.target.value)}
                            className="form-radio h-5 w-5 text-purple-600"
                        />
                        <span className="ml-2 text-gray-700">Lật thẻ (Flashcard Mode: Anh &rarr; Việt)</span>
                    </label>
                    <label className="inline-flex items-center">
                        <input
                            type="radio"
                            name="testMode"
                            value="eng-vie"
                            checked={testMode === 'eng-vie'}
                            onChange={(e) => setTestMode(e.target.value)}
                            className="form-radio h-5 w-5 text-purple-600"
                        />
                        <span className="ml-2 text-gray-700">Kiểm tra: Tiếng Anh &rarr; Tiếng Việt</span>
                    </label>
                    <label className="inline-flex items-center">
                        <input
                            type="radio"
                            name="testMode"
                            value="vie-eng"
                            checked={testMode === 'vie-eng'}
                            onChange={(e) => setTestMode(e.target.value)}
                            className="form-radio h-5 w-5 text-purple-600"
                        />
                        <span className="ml-2 text-gray-700">Kiểm tra: Tiếng Việt &rarr; Tiếng Anh</span>
                    </label>
                </div>
            </div>

            <div className="flex flex-col space-y-3">
                <button
                    onClick={() => startReviewSession(false)}
                    className="w-full px-6 py-3 bg-purple-600 text-white rounded-md font-semibold hover:bg-purple-700 transition-colors shadow-lg"
                >
                    Bắt đầu ôn tập (Bài mới)
                </button>
                {lastTestFlashcards.length > 0 && (
                    <button
                        onClick={() => startReviewSession(true)}
                        className="w-full px-6 py-3 bg-blue-600 text-white rounded-md font-semibold hover:bg-blue-700 transition-colors shadow-lg"
                    >
                        Làm lại bài kiểm tra cũ ({lastTestFlashcards.length} câu - Xáo trộn mới)
                    </button>
                )}
            </div>
        </div>
    );
};

// Component for the actual Test/Quiz
const Test = ({ flashcards, duration, mode, onTestEnd }) => {
    const [currentQuestionIndex, setCurrentQuestionIndex] = useState(0);
    const [score, setScore] = useState(0);
    const [timeLeft, setTimeLeft] = useState(duration);
    const [answer, setAnswer] = useState('');
    const [isCorrect, setIsCorrect] = useState(null); // null: no check, true: correct, false: incorrect
    const [showResults, setShowResults] = useState(false);
    const [hasCheated, setHasCheated] = useState(false);
    const timerRef = useRef(null);
    const answerInputRef = useRef(null);

    // Anti-cheat: Detect tab/window visibility change
    const handleVisibilityChange = useCallback(() => {
        if (document.hidden) {
            setHasCheated(true);
            clearInterval(timerRef.current);
            setShowResults(true); // End test immediately if cheating detected
        }
    }, []);

    useEffect(() => {
        document.addEventListener('visibilitychange', handleVisibilityChange);
        return () => {
            document.removeEventListener('visibilitychange', handleVisibilityChange);
        };
    }, [handleVisibilityChange]);


    // Start timer on test start
    useEffect(() => {
        if (!showResults && !hasCheated) {
            timerRef.current = setInterval(() => {
                setTimeLeft(prev => {
                    if (prev <= 1) {
                        clearInterval(timerRef.current);
                        setShowResults(true); // Time's up, show results
                        return 0;
                    }
                    return prev - 1;
                });
            }, 1000);
        }

        // Cleanup on unmount or test end
        return () => clearInterval(timerRef.current);
    }, [showResults, hasCheated]);

    // Focus input when question changes
    useEffect(() => {
        if (answerInputRef.current) {
            answerInputRef.current.focus();
        }
    }, [currentQuestionIndex]);

    const currentCard = flashcards[currentQuestionIndex];
    const questionText = mode === 'eng-vie' ? currentCard?.englishWord : currentCard?.vietnameseWord;
    const correctAnswer = mode === 'eng-vie' ? currentCard?.vietnameseWord : currentCard?.englishWord;

    const checkAnswer = () => {
        if (!currentCard) return;

        const userAnswer = answer.trim().toLowerCase();
        const correctFormattedAnswer = correctAnswer.trim().toLowerCase();

        if (userAnswer === correctFormattedAnswer) {
            setScore(prev => prev + 1);
            setIsCorrect(true);
        } else {
            setIsCorrect(false);
        }
        // Move to next question after a short delay to show feedback
        setTimeout(() => {
            handleNextQuestion();
        }, 1000);
    };

    const handleNextQuestion = () => {
        setAnswer('');
        setIsCorrect(null); // Reset feedback
        if (currentQuestionIndex < flashcards.length - 1) {
            setCurrentQuestionIndex(prev => prev + 1);
        } else {
            clearInterval(timerRef.current);
            setShowResults(true); // End of questions, show results
        }
    };

    const formatTime = (seconds) => {
        const minutes = Math.floor(seconds / 60);
        const remainingSeconds = seconds % 60;
        return `${minutes.toString().padStart(2, '0')}:${remainingSeconds.toString().padStart(2, '0')}`;
    };

    if (showResults) {
        return (
            <div className="flex flex-col items-center p-8 bg-blue-50 rounded-lg shadow-xl">
                <h2 className="text-3xl font-bold text-blue-800 mb-6">Kết quả kiểm tra</h2>
                {hasCheated ? (
                    <p className="text-xl text-red-600 font-semibold mb-4 animate-pulse">
                        ⚠️ Phát hiện gian lận! Bài kiểm tra đã bị dừng.
                    </p>
                ) : (
                    <p className="text-2xl text-green-700 font-semibold mb-4">
                        Bạn đã đúng {score} / {flashcards.length} câu.
                    </p>
                )}
                <button
                    onClick={onTestEnd}
                    className="px-8 py-4 bg-blue-600 text-white rounded-full font-semibold text-lg hover:bg-blue-700 transition-colors shadow-lg"
                >
                    Quay lại trang Ôn tập
                </button>
            </div>
        );
    }

    if (!currentCard) {
        return (
            <div className="text-center text-gray-600 text-xl">
                Không có flashcard nào được tải cho bài kiểm tra này.
            </div>
        );
    }

    return (
        <div className="flex flex-col items-center p-8 bg-gradient-to-br from-blue-100 to-purple-100 rounded-lg shadow-xl relative min-h-[400px]">
            <div className="absolute top-4 right-4 bg-white text-blue-700 px-4 py-2 rounded-full font-semibold text-lg shadow-md">
                Thời gian: {formatTime(timeLeft)}
            </div>
            <div className="absolute top-4 left-4 bg-white text-green-700 px-4 py-2 rounded-full font-semibold text-lg shadow-md">
                Điểm: {score} / {flashcards.length}
            </div>

            <p className="text-gray-600 mb-2 mt-12 text-center text-sm">
                Câu hỏi {currentQuestionIndex + 1} / {flashcards.length}
            </p>
            <div className="flashcard-display bg-white p-10 rounded-xl shadow-lg w-full max-w-lg text-center min-h-[180px] flex items-center justify-center mb-8 transform hover:scale-105 transition-transform duration-300">
                <p className="text-4xl font-bold text-blue-900 break-words max-w-full">
                    {questionText}
                </p>
            </div>

            <input
                ref={answerInputRef}
                type="text"
                placeholder="Nhập câu trả lời của bạn..."
                value={answer}
                onChange={(e) => setAnswer(e.target.value)}
                onKeyPress={(e) => e.key === 'Enter' && checkAnswer()}
                className="w-full max-w-md p-4 text-center border-2 border-blue-300 rounded-full text-xl focus:ring-4 focus:ring-blue-500 outline-none transition-all duration-200"
            />

            {isCorrect !== null && (
                <div className={`mt-4 px-6 py-2 rounded-full font-semibold text-white shadow-md ${isCorrect ? 'bg-green-500' : 'bg-red-500'}`}>
                    {isCorrect ? 'Chính xác!' : `Sai rồi! Đáp án: ${correctAnswer}`}
                </div>
            )}

            <div className="mt-8 flex space-x-4">
                <button
                    onClick={checkAnswer}
                    className="px-8 py-4 bg-blue-600 text-white rounded-full font-semibold text-lg hover:bg-blue-700 transition-colors shadow-lg transform hover:scale-105"
                >
                    Kiểm tra
                </button>
                <button
                    onClick={handleNextQuestion}
                    className="px-8 py-4 bg-gray-300 text-gray-800 rounded-full font-semibold text-lg hover:bg-gray-400 transition-colors shadow-lg transform hover:scale-105"
                >
                    Bỏ qua &rarr;
                </button>
            </div>
        </div>
    );
};

// New Component for Flip Card Mode
const FlipCardMode = ({ flashcards, onEnd }) => {
    const [currentIndex, setCurrentIndex] = useState(0);
    const [isFlipped, setIsFlipped] = useState(false);

    // Reset flip state when card changes
    useEffect(() => {
        setIsFlipped(false);
    }, [currentIndex]);

    const currentCard = flashcards[currentIndex];

    if (!currentCard) {
        return (
            <div className="flex flex-col items-center p-8 bg-blue-50 rounded-lg shadow-xl">
                <h2 className="text-2xl font-bold text-blue-800 mb-4">Kết thúc phiên lật thẻ</h2>
                <p className="text-xl text-gray-700 mb-6">Bạn đã xem hết {flashcards.length} thẻ.</p>
                <button
                    onClick={onEnd}
                    className="px-8 py-4 bg-blue-600 text-white rounded-full font-semibold text-lg hover:bg-blue-700 transition-colors shadow-lg"
                >
                    Quay lại trang Ôn tập
                </button>
            </div>
        );
    }

    // Hardcode front to English and back to Vietnamese for Flip Card Mode
    const frontText = currentCard.englishWord;
    const backText = currentCard.vietnameseWord;

    const handleFlip = () => {
        setIsFlipped(prev => !prev);
    };

    const handleNext = () => {
        if (currentIndex < flashcards.length - 1) {
            setCurrentIndex(prev => prev + 1);
        } else {
            onEnd(); // End session if it's the last card
        }
    };

    const handlePrev = () => {
        if (currentIndex > 0) {
            setCurrentIndex(prev => prev - 1);
        }
    };

    return (
        <div className="flex flex-col items-center p-8 bg-gradient-to-br from-indigo-100 to-teal-100 rounded-lg shadow-xl relative min-h-[400px]">
            <p className="text-gray-600 mb-6 text-center text-lg">
                Thẻ {currentIndex + 1} / {flashcards.length}
            </p>

            <div className="flip-card w-full max-w-md h-64 perspective-1000 mb-8">
                <div
                    className={`flip-card-inner relative w-full h-full text-center transition-transform duration-700 rounded-xl shadow-lg cursor-pointer ${isFlipped ? 'rotated' : ''}`}
                    onClick={handleFlip}
                >
                    <div className="flip-card-front absolute w-full h-full backface-hidden bg-white flex items-center justify-center p-6 rounded-xl border-4 border-indigo-300">
                        <p className="text-3xl font-bold text-indigo-800 break-words max-w-full">
                            {frontText}
                        </p>
                    </div>
                    <div className="flip-card-back absolute w-full h-full backface-hidden bg-white flex items-center justify-center p-6 rounded-xl border-4 border-teal-300 transform rotate-y-180">
                        <p className="text-3xl font-bold text-teal-800 break-words max-w-full">
                            {backText}
                        </p>
                    </div>
                </div>
            </div>

            <div className="flex space-x-4 mt-4">
                <button
                    onClick={handlePrev}
                    disabled={currentIndex === 0}
                    className={`px-6 py-3 rounded-full font-semibold text-lg transition-colors shadow-lg transform hover:scale-105 ${
                        currentIndex === 0 ? 'bg-gray-300 text-gray-500 cursor-not-allowed' : 'bg-indigo-600 text-white hover:bg-indigo-700'
                    }`}
                >
                    &larr; Thẻ trước
                </button>
                <button
                    onClick={handleNext}
                    className="px-6 py-3 bg-teal-600 text-white rounded-full font-semibold text-lg hover:bg-teal-700 transition-colors shadow-lg transform hover:scale-105"
                >
                    Thẻ tiếp theo &rarr;
                </button>
            </div>

            {/* CSS for flip card animation */}
            <style>{`
                .perspective-1000 {
                    perspective: 1000px;
                }
                .flip-card-inner {
                    transform-style: preserve-3d;
                }
                .flip-card-inner.rotated {
                    transform: rotateY(180deg);
                }
                .flip-card-front, .flip-card-back {
                    -webkit-backface-visibility: hidden; /* Safari */
                    backface-visibility: hidden;
                }
                .flip-card-back {
                    transform: rotateY(180deg);
                }
            `}</style>
        </div>
    );
};

export default App;
