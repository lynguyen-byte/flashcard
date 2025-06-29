import React, { useState, useEffect, createContext, useContext } from 'react';
import { initializeApp } from 'firebase/app';
import { getAuth, signInAnonymously, signInWithCustomToken, onAuthStateChanged, signOut } from 'firebase/auth';
import { getFirestore, collection, query, onSnapshot, addDoc, doc, updateDoc, deleteDoc, getDocs, where } from 'firebase/firestore';

// Tailwind CSS is assumed to be available.
// lucide-react for icons (optional, could use inline SVGs or Font Awesome)
// Install via npm: npm install lucide-react

// --- Firebase Initialization and Context ---
const FirebaseContext = createContext(null);

const FirebaseProvider = ({ children }) => {
    const [app, setApp] = useState(null);
    const [db, setDb] = useState(null);
    const [auth, setAuth] = useState(null);
    const [userId, setUserId] = useState(null);
    const [isAuthReady, setIsAuthReady] = useState(false); // New state to track auth readiness

    useEffect(() => {
        try {
            // Global variables provided by the Canvas environment
            const appId = typeof __app_id !== 'undefined' ? __app_id : 'default-app-id';
            const firebaseConfig = typeof __firebase_config !== 'undefined' ? JSON.parse(__firebase_config) : {};
            const initialAuthToken = typeof __initial_auth_token !== 'undefined' ? __initial_auth_token : null;

            if (Object.keys(firebaseConfig).length === 0) {
                console.error("Firebase config is missing. Please provide it in __firebase_config.");
                return;
            }

            const firebaseApp = initializeApp(firebaseConfig);
            const firestoreDb = getFirestore(firebaseApp);
            const firebaseAuth = getAuth(firebaseApp);

            setApp(firebaseApp);
            setDb(firestoreDb);
            setAuth(firebaseAuth);

            // Authentication listener
            const unsubscribe = onAuthStateChanged(firebaseAuth, async (user) => {
                if (user) {
                    setUserId(user.uid);
                    setIsAuthReady(true);
                } else {
                    // Sign in with custom token or anonymously if not authenticated
                    try {
                        if (initialAuthToken) {
                            await signInWithCustomToken(firebaseAuth, initialAuthToken);
                        } else {
                            await signInAnonymously(firebaseAuth);
                        }
                    } catch (error) {
                        console.error("Firebase authentication error:", error);
                        // Fallback to a random ID if auth fails completely
                        setUserId(crypto.randomUUID());
                        setIsAuthReady(true);
                    }
                }
            });

            return () => unsubscribe(); // Cleanup auth listener on unmount
        } catch (error) {
            console.error("Failed to initialize Firebase:", error);
        }
    }, []);

    // Ensure userId is set, even if auth fails, so other components can proceed with a random ID
    useEffect(() => {
        if (!userId && isAuthReady && auth && !auth.currentUser) {
            setUserId(crypto.randomUUID()); // Assign a random ID if not authenticated and auth is ready
        }
    }, [isAuthReady, userId, auth]);

    return (
        <FirebaseContext.Provider value={{ app, db, auth, userId, isAuthReady }}>
            {children}
        </FirebaseContext.Provider>
    );
};

const useFirebase = () => useContext(FirebaseContext);

// --- Reusable Modal Component ---
const Modal = ({ show, title, message, onClose, onConfirm, showConfirmButton = false }) => {
    if (!show) return null;

    return (
        <div className="fixed inset-0 bg-gray-600 bg-opacity-75 flex items-center justify-center z-50 p-4">
            <div className="bg-white rounded-lg shadow-xl p-6 w-full max-w-md">
                <h3 className="text-xl font-bold mb-4 text-gray-800">{title}</h3>
                <p className="text-gray-700 mb-6">{message}</p>
                <div className="flex justify-end space-x-3">
                    {showConfirmButton && (
                        <button
                            onClick={onConfirm}
                            className="px-5 py-2 rounded-lg bg-blue-600 text-white font-semibold hover:bg-blue-700 transition-colors"
                        >
                            Xác nhận
                        </button>
                    )}
                    <button
                        onClick={onClose}
                        className="px-5 py-2 rounded-lg bg-gray-200 text-gray-800 font-semibold hover:bg-gray-300 transition-colors"
                    >
                        Đóng
                    </button>
                </div>
            </div>
        </div>
    );
};

// --- Loading Spinner Component ---
const LoadingSpinner = () => (
    <div className="flex justify-center items-center h-full">
        <div className="animate-spin rounded-full h-12 w-12 border-b-2 border-blue-500"></div>
    </div>
);

// --- Main App Component ---
const App = () => {
    const { db, userId, isAuthReady } = useFirebase();
    const [topics, setTopics] = useState([]);
    const [vocabulary, setVocabulary] = useState([]);
    const [view, setView] = useState('add'); // Changed initial view from 'home' to 'add'
    const [modal, setModal] = useState({ show: false, title: '', message: '', onConfirm: null, showConfirmButton: false });
    const [isLoading, setIsLoading] = useState(true);

    // Fetch topics and vocabulary when Firebase is ready and userId is available
    useEffect(() => {
        if (db && userId && isAuthReady) {
            setIsLoading(true);
            // Fetch topics
            const topicsCollectionRef = collection(db, `artifacts/${__app_id}/users/${userId}/topics`);
            const unsubscribeTopics = onSnapshot(topicsCollectionRef, (snapshot) => {
                const fetchedTopics = snapshot.docs.map(doc => ({ id: doc.id, ...doc.data() }));
                setTopics(fetchedTopics);
            }, (error) => {
                console.error("Error fetching topics:", error);
                setModal({
                    show: true,
                    title: "Lỗi",
                    message: "Không thể tải chủ đề. Vui lòng thử lại sau.",
                    onClose: () => setModal({ ...modal, show: false })
                });
            });

            // Fetch vocabulary
            const vocabularyCollectionRef = collection(db, `artifacts/${__app_id}/users/${userId}/vocabulary`);
            const unsubscribeVocabulary = onSnapshot(vocabularyCollectionRef, (snapshot) => {
                const fetchedVocabulary = snapshot.docs.map(doc => ({ id: doc.id, ...doc.data() }));
                setVocabulary(fetchedVocabulary);
                setIsLoading(false);
            }, (error) => {
                console.error("Error fetching vocabulary:", error);
                setModal({
                    show: true,
                    title: "Lỗi",
                    message: "Không thể tải từ vựng. Vui lòng thử lại sau.",
                    onClose: () => setModal({ ...modal, show: false })
                });
                setIsLoading(false);
            });

            return () => {
                unsubscribeTopics();
                unsubscribeVocabulary();
            };
        }
    }, [db, userId, isAuthReady]);

    // Handle adding a new topic
    const handleAddTopic = async (topicName) => {
        if (!db || !userId) {
            setModal({
                show: true,
                title: "Lỗi",
                message: "Hệ thống chưa sẵn sàng. Vui lòng đợi hoặc thử lại.",
                onClose: () => setModal({ ...modal, show: false })
            });
            return;
        }
        if (topics.some(t => t.name.toLowerCase() === topicName.toLowerCase())) {
            setModal({
                show: true,
                title: "Thông báo",
                message: `Chủ đề "${topicName}" đã tồn tại.`,
                onClose: () => setModal({ ...modal, show: false })
            });
            return;
        }
        try {
            await addDoc(collection(db, `artifacts/${__app_id}/users/${userId}/topics`), {
                name: topicName,
                createdAt: new Date()
            });
            setModal({
                show: true,
                title: "Thành công",
                message: `Đã thêm chủ đề "${topicName}".`,
                onClose: () => setModal({ ...modal, show: false })
            });
        } catch (e) {
            console.error("Error adding topic: ", e);
            setModal({
                show: true,
                title: "Lỗi",
                message: "Không thể thêm chủ đề. Vui lòng thử lại.",
                onClose: () => setModal({ ...modal, show: false })
            });
        }
    };

    // Handle adding a new flashcard
    const handleAddFlashcard = async (english, vietnamese, topicId) => {
        if (!db || !userId) {
            setModal({
                show: true,
                title: "Lỗi",
                message: "Hệ thống chưa sẵn sàng. Vui lòng đợi hoặc thử lại.",
                onClose: () => setModal({ ...modal, show: false })
            });
            return;
        }
        try {
            await addDoc(collection(db, `artifacts/${__app_id}/users/${userId}/vocabulary`), {
                english: english,
                vietnamese: vietnamese,
                topicId: topicId,
                createdAt: new Date()
            });
            setModal({
                show: true,
                title: "Thành công",
                message: `Đã thêm từ: ${english} - ${vietnamese}`,
                onClose: () => setModal({ ...modal, show: false })
            });
        } catch (e) {
            console.error("Error adding flashcard: ", e);
            setModal({
                show: true,
                title: "Lỗi",
                message: "Không thể thêm từ vựng. Vui lòng thử lại.",
                onClose: () => setModal({ ...modal, show: false })
            });
        }
    };

    // Handle bulk import
    const handleBulkImport = async (text, topicId) => {
        if (!db || !userId) {
            setModal({
                show: true,
                title: "Lỗi",
                message: "Hệ thống chưa sẵn sàng. Vui lòng đợi hoặc thử lại.",
                onClose: () => setModal({ ...modal, show: false })
            });
            return;
        }

        const lines = text.split('\n').filter(line => line.trim() !== '');
        let importedCount = 0;
        let failedCount = 0;

        for (const line of lines) {
            const parts = line.split(/[-:]/).map(part => part.trim()); // Split by hyphen or colon
            if (parts.length >= 2 && parts[0] && parts[1]) {
                const english = parts[0];
                const vietnamese = parts[1];
                try {
                    await addDoc(collection(db, `artifacts/${__app_id}/users/${userId}/vocabulary`), {
                        english: english,
                        vietnamese: vietnamese,
                        topicId: topicId,
                        createdAt: new Date()
                    });
                    importedCount++;
                } catch (e) {
                    console.error("Error adding bulk flashcard: ", line, e);
                    failedCount++;
                }
            } else {
                failedCount++;
            }
        }
        setModal({
            show: true,
            title: "Kết quả nhập hàng loạt",
            message: `Đã nhập thành công ${importedCount} từ. Thất bại ${failedCount} từ.`,
            onClose: () => setModal({ ...modal, show: false })
        });
    };


    if (isLoading) {
        return (
            <div className="min-h-screen bg-gray-50 flex flex-col items-center justify-center p-4">
                <LoadingSpinner />
                <p className="mt-4 text-gray-600">Đang tải dữ liệu và xác thực...</p>
            </div>
        );
    }

    return (
        <div className="min-h-screen bg-gradient-to-br from-blue-50 to-indigo-100 flex flex-col font-inter">
            <header className="bg-white shadow-md p-4 sticky top-0 z-40">
                <nav className="container mx-auto flex flex-col sm:flex-row justify-between items-center">
                    <h1 className="text-3xl font-extrabold text-blue-700 mb-2 sm:mb-0">Flashcard App</h1>
                    <div className="flex space-x-3">
                        {/* Removed NavLink for 'home' */}
                        <NavLink currentView={view} targetView="add" setView={setView}>Thêm từ</NavLink>
                        <NavLink currentView={view} targetView="quiz" setView={setView}>Kiểm tra</NavLink>
                        <NavLink currentView={view} targetView="study" setView={setView}>Học nhanh</NavLink>
                    </div>
                </nav>
            </header>

            <main className="flex-grow container mx-auto p-4 py-8">
                <div className="text-right text-sm text-gray-600 mb-4">
                    ID người dùng của bạn: <span className="font-mono bg-gray-100 p-1 rounded-md">{userId || 'Đang tải...'}</span>
                </div>

                {/* Removed HomeView rendering */}
                {view === 'add' && (
                    <FlashcardInputForm
                        topics={topics}
                        onAddTopic={handleAddTopic}
                        onAddFlashcard={handleAddFlashcard}
                        onBulkImport={handleBulkImport}
                    />
                )}
                {view === 'quiz' && (
                    <QuizComponent
                        vocabulary={vocabulary}
                        topics={topics}
                        db={db}
                        userId={userId}
                        appId={__app_id} // Pass appId for quiz history
                        setModal={setModal}
                    />
                )}
                {view === 'study' && (
                    <StudyModeComponent
                        vocabulary={vocabulary}
                        topics={topics}
                        setModal={setModal}
                    />
                )}
            </main>

            <footer className="bg-gray-800 text-white p-4 mt-8">
                <div className="container mx-auto text-center text-sm">
                    © 2024 Flashcard App. Powered by React & Firebase.
                </div>
            </footer>

            <Modal
                show={modal.show}
                title={modal.title}
                message={modal.message}
                onClose={() => setModal({ ...modal, show: false })}
                onConfirm={modal.onConfirm}
                showConfirmButton={modal.showConfirmButton}
            />
        </div>
    );
};

// --- Navigation Link Component ---
const NavLink = ({ currentView, targetView, setView, children }) => (
    <button
        onClick={() => setView(targetView)}
        className={`px-4 py-2 rounded-lg font-medium transition-colors duration-200
            ${currentView === targetView
                ? 'bg-blue-600 text-white shadow-md'
                : 'text-gray-700 hover:bg-gray-100'
            }`}
    >
        {children}
    </button>
);

// --- Home View Component ---
// This component is no longer used but kept for reference if needed later.
const HomeView = ({ setView, vocabularyCount, topicCount }) => (
    <section className="bg-white rounded-xl shadow-xl p-8 max-w-2xl mx-auto text-center animate-fade-in">
        <h2 className="text-4xl font-extrabold text-blue-800 mb-6">Chào mừng đến với Flashcard App!</h2>
        <p className="text-lg text-gray-700 mb-8 leading-relaxed">
            Học từ vựng tiếng Anh chưa bao giờ dễ dàng đến thế. Quản lý các bộ flashcard của bạn, kiểm tra bản thân và theo dõi tiến độ.
        </p>
        <div className="grid grid-cols-1 md:grid-cols-2 gap-6 mb-8">
            <div className="bg-blue-50 p-6 rounded-lg shadow-inner">
                <h3 className="text-xl font-semibold text-blue-700">Tổng số từ vựng</h3>
                <p className="text-5xl font-bold text-blue-900 mt-2">{vocabularyCount}</p>
            </div>
            <div className="bg-indigo-50 p-6 rounded-lg shadow-inner">
                <h3 className="text-xl font-semibold text-indigo-700">Tổng số chủ đề</h3>
                <p className="text-5xl font-bold text-indigo-900 mt-2">{topicCount}</p>
            </div>
        </div>
        <div className="flex flex-col sm:flex-row justify-center space-y-4 sm:space-y-0 sm:space-x-4">
            <button
                onClick={() => setView('add')}
                className="bg-green-600 text-white px-8 py-3 rounded-lg text-lg font-semibold hover:bg-green-700 transition-transform transform hover:scale-105 shadow-lg"
            >
                Thêm từ vựng mới
            </button>
            <button
                onClick={() => setView('quiz')}
                className="bg-purple-600 text-white px-8 py-3 rounded-lg text-lg font-semibold hover:bg-purple-700 transition-transform transform hover:scale-105 shadow-lg"
            >
                Bắt đầu kiểm tra
            </button>
        </div>
    </section>
);


// --- Flashcard Input Form Component ---
const FlashcardInputForm = ({ topics, onAddTopic, onAddFlashcard, onBulkImport }) => {
    const [englishWord, setEnglishWord] = useState('');
    const [vietnameseWord, setVietnameseWord] = useState('');
    const [selectedTopic, setSelectedTopic] = useState(''); // Stores topic ID
    const [newTopicName, setNewTopicName] = useState('');
    const [bulkText, setBulkText] = useState('');
    const [bulkImportTopic, setBulkImportTopic] = useState(''); // Stores topic ID for bulk import

    const handleSubmitFlashcard = (e) => {
        e.preventDefault();
        if (englishWord.trim() && vietnameseWord.trim() && selectedTopic) {
            onAddFlashcard(englishWord.trim(), vietnameseWord.trim(), selectedTopic);
            setEnglishWord('');
            setVietnameseWord('');
        } else {
            // Use a custom modal instead of alert
            // Placeholder: alert('Vui lòng điền đầy đủ từ tiếng Anh, tiếng Việt và chọn chủ đề.');
        }
    };

    const handleAddNewTopic = (e) => {
        e.preventDefault();
        if (newTopicName.trim()) {
            onAddTopic(newTopicName.trim());
            setNewTopicName('');
        } else {
            // Placeholder: alert('Vui lòng nhập tên chủ đề mới.');
        }
    };

    const handleBulkSubmit = (e) => {
        e.preventDefault();
        if (bulkText.trim() && bulkImportTopic) {
            onBulkImport(bulkText.trim(), bulkImportTopic);
            setBulkText('');
        } else {
            // Placeholder: alert('Vui lòng nhập văn bản và chọn chủ đề cho nhập hàng loạt.');
        }
    };

    return (
        <section className="bg-white rounded-xl shadow-xl p-8 max-w-4xl mx-auto grid grid-cols-1 lg:grid-cols-2 gap-8">
            {/* Thêm từ vựng đơn lẻ */}
            <div className="border-r border-gray-200 lg:pr-8">
                <h2 className="text-3xl font-bold text-blue-700 mb-6">Thêm từ vựng mới</h2>
                <form onSubmit={handleSubmitFlashcard} className="space-y-4">
                    <div>
                        <label htmlFor="englishWord" className="block text-gray-700 text-sm font-semibold mb-2">
                            Từ tiếng Anh:
                        </label>
                        <input
                            type="text"
                            id="englishWord"
                            className="w-full px-4 py-2 border border-gray-300 rounded-lg focus:outline-none focus:ring-2 focus:ring-blue-500"
                            value={englishWord}
                            onChange={(e) => setEnglishWord(e.target.value)}
                            required
                        />
                    </div>
                    <div>
                        <label htmlFor="vietnameseWord" className="block text-gray-700 text-sm font-semibold mb-2">
                            Nghĩa tiếng Việt:
                        </label>
                        <input
                            type="text"
                            id="vietnameseWord"
                            className="w-full px-4 py-2 border border-gray-300 rounded-lg focus:outline-none focus:ring-2 focus:ring-blue-500"
                            value={vietnameseWord}
                            onChange={(e) => setVietnameseWord(e.target.value)}
                            required
                        />
                    </div>
                    <div>
                        <label htmlFor="selectTopic" className="block text-gray-700 text-sm font-semibold mb-2">
                            Chọn chủ đề:
                        </label>
                        <select
                            id="selectTopic"
                            className="w-full px-4 py-2 border border-gray-300 rounded-lg bg-white focus:outline-none focus:ring-2 focus:ring-blue-500"
                            value={selectedTopic}
                            onChange={(e) => setSelectedTopic(e.target.value)}
                            required
                        >
                            <option value="">-- Chọn chủ đề --</option>
                            {topics.map((topic) => (
                                <option key={topic.id} value={topic.id}>
                                    {topic.name}
                                </option>
                            ))}
                        </select>
                    </div>
                    <button
                        type="submit"
                        className="w-full bg-blue-600 text-white py-3 rounded-lg font-semibold text-lg hover:bg-blue-700 transition-colors shadow-md"
                    >
                        Thêm Flashcard
                    </button>
                </form>

                {/* Thêm chủ đề mới */}
                <div className="mt-8 pt-6 border-t border-gray-200">
                    <h3 className="text-2xl font-bold text-blue-700 mb-4">Hoặc tạo chủ đề mới</h3>
                    <form onSubmit={handleAddNewTopic} className="flex space-x-2">
                        <input
                            type="text"
                            className="flex-grow px-4 py-2 border border-gray-300 rounded-lg focus:outline-none focus:ring-2 focus:ring-blue-500"
                            placeholder="Tên chủ đề mới"
                            value={newTopicName}
                            onChange={(e) => setNewTopicName(e.target.value)}
                            required
                        />
                        <button
                            type="submit"
                            className="bg-indigo-600 text-white px-6 py-2 rounded-lg font-semibold hover:bg-indigo-700 transition-colors shadow-md"
                        >
                            Tạo chủ đề
                        </button>
                    </form>
                </div>
            </div>

            {/* Nhập hàng loạt */}
            <div className="lg:pl-8">
                <h2 className="text-3xl font-bold text-blue-700 mb-6">Nhập hàng loạt từ văn bản</h2>
                <p className="text-gray-600 mb-4 text-sm">
                    Dán văn bản từ vựng của bạn vào đây. Mỗi cặp từ tiếng Anh-Việt trên một dòng, phân cách bởi dấu gạch ngang (-) hoặc dấu hai chấm (:).
                    <br />Ví dụ: <code className="bg-gray-100 p-1 rounded text-xs">hello - xin chào</code> hoặc <code className="bg-gray-100 p-1 rounded text-xs">book: sách</code>
                </p>
                <form onSubmit={handleBulkSubmit} className="space-y-4">
                    <div>
                        <label htmlFor="bulkText" className="block text-gray-700 text-sm font-semibold mb-2">
                            Văn bản từ vựng:
                        </label>
                        <textarea
                            id="bulkText"
                            className="w-full px-4 py-2 border border-gray-300 rounded-lg h-40 resize-y focus:outline-none focus:ring-2 focus:ring-blue-500"
                            value={bulkText}
                            onChange={(e) => setBulkText(e.target.value)}
                            placeholder="Nhập từng cặp từ trên mỗi dòng, ví dụ: &#10;apple - quả táo&#10;banana : quả chuối"
                            required
                        ></textarea>
                    </div>
                    <div>
                        <label htmlFor="bulkImportTopic" className="block text-gray-700 text-sm font-semibold mb-2">
                            Chọn chủ đề cho nhập hàng loạt:
                        </label>
                        <select
                            id="bulkImportTopic"
                            className="w-full px-4 py-2 border border-gray-300 rounded-lg bg-white focus:outline-none focus:ring-2 focus:ring-blue-500"
                            value={bulkImportTopic}
                            onChange={(e) => setBulkImportTopic(e.target.value)}
                            required
                        >
                            <option value="">-- Chọn chủ đề --</option>
                            {topics.map((topic) => (
                                <option key={topic.id} value={topic.id}>
                                    {topic.name}
                                </option>
                            ))}
                        </select>
                    </div>
                    <button
                        type="submit"
                        className="w-full bg-green-600 text-white py-3 rounded-lg font-semibold text-lg hover:bg-green-700 transition-colors shadow-md"
                    >
                        Nhập hàng loạt
                    </button>
                    <p className="text-sm text-gray-500 mt-2">
                        Lưu ý: Tính năng nhập từ hình ảnh (OCR) yêu cầu các dịch vụ backend chuyên biệt. Tại đây, bạn có thể dán văn bản đã được trích xuất từ hình ảnh.
                    </p>
                </form>
            </div>
        </section>
    );
};


// --- Quiz Component ---
const QuizComponent = ({ vocabulary, topics, db, userId, appId, setModal }) => {
    const [quizSettings, setQuizSettings] = useState({
        topicId: '',
        direction: 'en-vi', // 'en-vi' or 'vi-en'
        timeLimit: 0, // seconds, 0 for no limit
        quizLength: 10,
    });
    const [currentQuiz, setCurrentQuiz] = useState([]); // Questions for the current quiz session
    const [currentQuestionIndex, setCurrentQuestionIndex] = useState(0);
    const [userAnswer, setUserAnswer] = useState('');
    const [quizStarted, setQuizStarted] = useState(false);
    const [quizFinished, setQuizFinished] = useState(false);
    const [score, setScore] = useState(0);
    const [timer, setTimer] = useState(0);
    const [intervalId, setIntervalId] = useState(null);
    const [showCorrectAnswer, setShowCorrectAnswer] = useState(false);
    const [quizHistory, setQuizHistory] = useState([]); // To store previous quiz sessions
    const [selectedHistoryQuizId, setSelectedHistoryQuizId] = useState('');
    const [isReviewMode, setIsReviewMode] = useState(false);

    // Fetch quiz history
    useEffect(() => {
        if (db && userId) {
            const historyCollectionRef = collection(db, `artifacts/${appId}/users/${userId}/quizHistory`);
            const q = query(historyCollectionRef); // No orderBy to avoid index issues
            const unsubscribe = onSnapshot(q, (snapshot) => {
                const fetchedHistory = snapshot.docs.map(doc => ({ id: doc.id, ...doc.data() }));
                // Sort by date in memory
                fetchedHistory.sort((a, b) => (b.timestamp?.toDate() || 0) - (a.timestamp?.toDate() || 0));
                setQuizHistory(fetchedHistory);
            }, (error) => {
                console.error("Error fetching quiz history:", error);
            });
            return () => unsubscribe();
        }
    }, [db, userId, appId]);

    // Timer effect
    useEffect(() => {
        if (quizStarted && quizSettings.timeLimit > 0 && !quizFinished) {
            setTimer(quizSettings.timeLimit);
            const id = setInterval(() => {
                setTimer((prev) => {
                    if (prev <= 1) {
                        clearInterval(id);
                        handleNextQuestion(true); // Auto-submit if time runs out
                        return 0;
                    }
                    return prev - 1;
                });
            }, 1000);
            setIntervalId(id);
            return () => clearInterval(id); // Cleanup on unmount or if quiz stops
        } else if (!quizStarted || quizFinished) {
            clearInterval(intervalId);
        }
    }, [quizStarted, currentQuestionIndex, quizFinished, quizSettings.timeLimit]);

    // Shuffle array function
    const shuffleArray = (array) => {
        let shuffled = [...array];
        for (let i = shuffled.length - 1; i > 0; i--) {
            const j = Math.floor(Math.random() * (i + 1));
            [shuffled[i], shuffled[j]] = [shuffled[j], shuffled[i]];
        }
        return shuffled;
    };

    // Prepare questions based on settings
    const prepareQuestions = () => {
        let filteredVocabulary = vocabulary;
        if (quizSettings.topicId) {
            filteredVocabulary = vocabulary.filter(voc => voc.topicId === quizSettings.topicId);
        }

        if (filteredVocabulary.length === 0) {
            setModal({
                show: true,
                title: "Không có từ vựng",
                message: "Không có từ vựng nào phù hợp với cài đặt của bạn. Vui lòng thêm từ hoặc chọn chủ đề khác.",
                onClose: () => setModal({ ...setModal, show: false })
            });
            return [];
        }

        let shuffledVocabulary = shuffleArray(filteredVocabulary);
        // Ensure quizLength doesn't exceed available vocabulary
        const actualQuizLength = Math.min(quizSettings.quizLength, shuffledVocabulary.length);
        return shuffledVocabulary.slice(0, actualQuizLength).map(item => ({
            ...item,
            isCorrect: false, // Track if answer was correct
            userAnswer: '', // Store user's answer
            question: quizSettings.direction === 'en-vi' ? item.english : item.vietnamese,
            correctAnswer: quizSettings.direction === 'en-vi' ? item.vietnamese : item.english
        }));
    };

    // Start a new quiz
    const startQuiz = () => {
        setIsReviewMode(false);
        const questions = prepareQuestions();
        if (questions.length > 0) {
            setCurrentQuiz(questions);
            setCurrentQuestionIndex(0);
            setScore(0);
            setUserAnswer('');
            setShowCorrectAnswer(false);
            setQuizFinished(false);
            setQuizStarted(true);
            if (quizSettings.timeLimit > 0) {
                setTimer(quizSettings.timeLimit);
            }
        } else {
            setQuizStarted(false);
        }
    };

    // Start a review quiz from history
    const startReviewQuiz = () => {
        const selectedQuiz = quizHistory.find(q => q.id === selectedHistoryQuizId);
        if (selectedQuiz && selectedQuiz.questions) {
            const reviewQuestions = shuffleArray(selectedQuiz.questions.map(q => ({
                ...q,
                isCorrect: false, // Reset correctness for review
                userAnswer: '', // Reset user answer for review
                question: quizSettings.direction === 'en-vi' ? q.english : q.vietnamese,
                correctAnswer: quizSettings.direction === 'en-vi' ? q.vietnamese : q.english
            })));
            setCurrentQuiz(reviewQuestions);
            setCurrentQuestionIndex(0);
            setScore(0);
            setUserAnswer('');
            setShowCorrectAnswer(false);
            setQuizFinished(false);
            setQuizStarted(true);
            setIsReviewMode(true);
            if (quizSettings.timeLimit > 0) {
                setTimer(quizSettings.timeLimit);
            }
        } else {
            setModal({
                show: true,
                title: "Lỗi",
                message: "Không tìm thấy bộ câu hỏi để ôn tập.",
                onClose: () => setModal({ ...setModal, show: false })
            });
        }
    };

    // Handle user submitting an answer or time running out
    const handleNextQuestion = (fromTimer = false) => {
        clearInterval(intervalId); // Stop current question's timer

        const currentQuestion = currentQuiz[currentQuestionIndex];
        const isAnswerCorrect = currentQuestion.correctAnswer.toLowerCase() === userAnswer.trim().toLowerCase();

        // Update the current quiz state with user's answer and correctness
        const updatedQuiz = currentQuiz.map((q, index) => {
            if (index === currentQuestionIndex) {
                return { ...q, userAnswer: fromTimer ? '' : userAnswer.trim(), isCorrect: isAnswerCorrect };
            }
            return q;
        });
        setCurrentQuiz(updatedQuiz);

        if (isAnswerCorrect) {
            setScore(prev => prev + 1);
        }

        setShowCorrectAnswer(true); // Show the correct answer after submission

        // Wait a bit before moving to the next question
        setTimeout(() => {
            if (currentQuestionIndex < currentQuiz.length - 1) {
                setCurrentQuestionIndex(prev => prev + 1);
                setUserAnswer('');
                setShowCorrectAnswer(false);
                if (quizSettings.timeLimit > 0) {
                    setTimer(quizSettings.timeLimit); // Reset timer for next question
                }
            } else {
                setQuizFinished(true);
                setQuizStarted(false);
                // Save quiz results to history if it's a new quiz (not a review)
                if (!isReviewMode && currentQuiz.length > 0) {
                    saveQuizHistory(updatedQuiz, score + (isAnswerCorrect ? 1 : 0)); // Final score after last question
                }
            }
        }, 1500); // Show correct answer for 1.5 seconds
    };

    // Save quiz history to Firestore
    const saveQuizHistory = async (quizData, finalScore) => {
        if (!db || !userId) return;

        try {
            await addDoc(collection(db, `artifacts/${appId}/users/${userId}/quizHistory`), {
                timestamp: new Date(),
                settings: quizSettings,
                questions: quizData.map(q => ({ // Store relevant info for review
                    english: q.english,
                    vietnamese: q.vietnamese,
                    topicId: q.topicId,
                    question: q.question,
                    correctAnswer: q.correctAnswer,
                    wasCorrect: q.isCorrect, // Store correctness from this quiz session
                    userAnswer: q.userAnswer,
                })),
                score: finalScore,
                totalQuestions: quizData.length,
            });
            setModal({
                show: true,
                title: "Lịch sử kiểm tra",
                message: "Kết quả kiểm tra đã được lưu vào lịch sử.",
                onClose: () => setModal({ ...setModal, show: false })
            });
        } catch (e) {
            console.error("Error saving quiz history: ", e);
            setModal({
                show: true,
                title: "Lỗi",
                message: "Không thể lưu lịch sử kiểm tra.",
                onClose: () => setModal({ ...setModal, show: false })
            });
        }
    };


    const currentQuestion = currentQuiz[currentQuestionIndex];
    const topicName = topics.find(t => t.id === quizSettings.topicId)?.name || 'Tất cả chủ đề';

    if (!quizStarted && !quizFinished) {
        return (
            <section className="bg-white rounded-xl shadow-xl p-8 max-w-2xl mx-auto">
                <h2 className="text-3xl font-bold text-blue-700 mb-6">Cài đặt kiểm tra</h2>
                <div className="space-y-4 mb-6">
                    <div>
                        <label htmlFor="quizTopic" className="block text-gray-700 text-sm font-semibold mb-2">
                            Chọn chủ đề:
                        </label>
                        <select
                            id="quizTopic"
                            className="w-full px-4 py-2 border border-gray-300 rounded-lg bg-white focus:outline-none focus:ring-2 focus:ring-blue-500"
                            value={quizSettings.topicId}
                            onChange={(e) => setQuizSettings({ ...quizSettings, topicId: e.target.value })}
                        >
                            <option value="">Tất cả chủ đề</option>
                            {topics.map(topic => (
                                <option key={topic.id} value={topic.id}>{topic.name}</option>
                            ))}
                        </select>
                    </div>
                    <div>
                        <label htmlFor="quizDirection" className="block text-gray-700 text-sm font-semibold mb-2">
                            Hướng kiểm tra:
                        </label>
                        <select
                            id="quizDirection"
                            className="w-full px-4 py-2 border border-gray-300 rounded-lg bg-white focus:outline-none focus:ring-2 focus:ring-blue-500"
                            value={quizSettings.direction}
                            onChange={(e) => setQuizSettings({ ...quizSettings, direction: e.target.value })}
                        >
                            <option value="en-vi">Anh sang Việt</option>
                            <option value="vi-en">Việt sang Anh</option>
                        </select>
                    </div>
                    <div>
                        <label htmlFor="quizTimeLimit" className="block text-gray-700 text-sm font-semibold mb-2">
                            Thời gian trả lời mỗi câu (giây, 0 = không giới hạn):
                        </label>
                        <input
                            type="number"
                            id="quizTimeLimit"
                            className="w-full px-4 py-2 border border-gray-300 rounded-lg focus:outline-none focus:ring-2 focus:ring-blue-500"
                            value={quizSettings.timeLimit}
                            onChange={(e) => setQuizSettings({ ...quizSettings, timeLimit: Math.max(0, parseInt(e.target.value) || 0) })}
                            min="0"
                        />
                    </div>
                    <div>
                        <label htmlFor="quizLength" className="block text-gray-700 text-sm font-semibold mb-2">
                            Số lượng câu hỏi:
                        </label>
                        <input
                            type="number"
                            id="quizLength"
                            className="w-full px-4 py-2 border border-gray-300 rounded-lg focus:outline-none focus:ring-2 focus:ring-blue-500"
                            value={quizSettings.quizLength}
                            onChange={(e) => setQuizSettings({ ...quizSettings, quizLength: Math.max(1, parseInt(e.target.value) || 1) })}
                            min="1"
                            max={vocabulary.length || 1}
                        />
                        <p className="text-sm text-gray-500 mt-1">
                            (Tổng số từ có sẵn: {vocabulary.length})
                        </p>
                    </div>
                </div>
                <button
                    onClick={startQuiz}
                    className="w-full bg-blue-600 text-white py-3 rounded-lg font-semibold text-lg hover:bg-blue-700 transition-colors shadow-md"
                >
                    Bắt đầu kiểm tra mới
                </button>

                <div className="mt-8 pt-6 border-t border-gray-200">
                    <h3 className="text-2xl font-bold text-blue-700 mb-4">Lịch sử kiểm tra</h3>
                    {quizHistory.length === 0 ? (
                        <p className="text-gray-600">Bạn chưa có lịch sử kiểm tra nào.</p>
                    ) : (
                        <div>
                            <select
                                className="w-full px-4 py-2 border border-gray-300 rounded-lg bg-white focus:outline-none focus:ring-2 focus:ring-blue-500 mb-4"
                                value={selectedHistoryQuizId}
                                onChange={(e) => setSelectedHistoryQuizId(e.target.value)}
                            >
                                <option value="">-- Chọn bài kiểm tra để ôn tập --</option>
                                {quizHistory.map(quiz => (
                                    <option key={quiz.id} value={quiz.id}>
                                        {new Date(quiz.timestamp?.toDate()).toLocaleString()} - {quiz.score}/{quiz.totalQuestions} ({topics.find(t => t.id === quiz.settings.topicId)?.name || 'Tổng hợp'})
                                    </option>
                                ))}
                            </select>
                            <button
                                onClick={startReviewQuiz}
                                disabled={!selectedHistoryQuizId}
                                className={`w-full py-3 rounded-lg font-semibold text-lg transition-colors shadow-md
                                    ${selectedHistoryQuizId ? 'bg-purple-600 text-white hover:bg-purple-700' : 'bg-gray-300 text-gray-500 cursor-not-allowed'}`}
                            >
                                Bắt đầu ôn tập
                            </button>
                        </div>
                    )}
                </div>
            </section>
        );
    }

    if (quizFinished) {
        return (
            <section className="bg-white rounded-xl shadow-xl p-8 max-w-2xl mx-auto text-center">
                <h2 className="text-3xl font-bold text-blue-700 mb-6">Kết quả kiểm tra!</h2>
                <p className="text-xl text-gray-800 mb-4">
                    Bạn đã trả lời đúng <span className="font-extrabold text-green-600">{score}</span> trên tổng số <span className="font-extrabold text-blue-600">{currentQuiz.length}</span> câu hỏi.
                </p>
                <div className="space-y-4 mb-6 text-left">
                    <h3 className="text-2xl font-bold text-blue-700 mb-4">Chi tiết:</h3>
                    {currentQuiz.map((q, index) => (
                        <div key={index} className={`p-4 rounded-lg shadow-sm ${q.isCorrect ? 'bg-green-50' : 'bg-red-50'}`}>
                            <p className="text-gray-800 font-semibold mb-1">
                                Câu hỏi {index + 1}: <span className="font-bold text-lg text-blue-800">{q.question}</span>
                            </p>
                            <p className="text-gray-700 mb-1">
                                Câu trả lời của bạn: <span className={`${q.isCorrect ? 'text-green-600' : 'text-red-600'} font-medium`}>
                                    {q.userAnswer || (q.isCorrect ? '' : '(Không trả lời)')}
                                </span>
                            </p>
                            {!q.isCorrect && (
                                <p className="text-gray-700">
                                    Đáp án đúng: <span className="text-green-700 font-medium">{q.correctAnswer}</span>
                                </p>
                            )}
                        </div>
                    ))}
                </div>
                <button
                    onClick={() => {
                        setQuizStarted(false);
                        setQuizFinished(false);
                        setCurrentQuiz([]);
                        setSelectedHistoryQuizId(''); // Clear selection
                        setIsReviewMode(false);
                    }}
                    className="w-full bg-blue-600 text-white py-3 rounded-lg font-semibold text-lg hover:bg-blue-700 transition-colors shadow-md"
                >
                    Kiểm tra lại
                </button>
            </section>
        );
    }

    return (
        <section className="bg-white rounded-xl shadow-xl p-8 max-w-2xl mx-auto text-center relative">
            <h2 className="text-3xl font-bold text-blue-700 mb-4">Đang kiểm tra</h2>
            <p className="text-lg text-gray-700 mb-2">
                Chủ đề: <span className="font-semibold">{topicName}</span> | Hướng: <span className="font-semibold">{quizSettings.direction === 'en-vi' ? 'Anh → Việt' : 'Việt → Anh'}</span>
            </p>
            <p className="text-xl text-gray-800 mb-6">
                Câu hỏi {currentQuestionIndex + 1} / {currentQuiz.length}
                {quizSettings.timeLimit > 0 && (
                    <span className="ml-4 text-red-600 font-bold">Thời gian: {timer}s</span>
                )}
            </p>

            {currentQuestion && (
                <>
                    <div className="flashcard-container mb-8">
                        <div className="flashcard-content bg-blue-100 p-8 rounded-lg shadow-lg">
                            <p className="text-4xl font-extrabold text-blue-900 mb-4 break-words">
                                {currentQuestion.question}
                            </p>
                            {showCorrectAnswer && (
                                <p className="text-3xl font-semibold text-green-700 break-words mt-4 animate-fade-in-down">
                                    {currentQuestion.correctAnswer}
                                </p>
                            )}
                        </div>
                    </div>

                    <form onSubmit={(e) => { e.preventDefault(); handleNextQuestion(); }} className="space-y-4">
                        <input
                            type="text"
                            className="w-full px-5 py-3 border-2 border-gray-300 rounded-lg text-lg focus:outline-none focus:ring-2 focus:ring-blue-500 disabled:bg-gray-100"
                            placeholder="Nhập câu trả lời của bạn..."
                            value={userAnswer}
                            onChange={(e) => setUserAnswer(e.target.value)}
                            disabled={showCorrectAnswer} // Disable input while correct answer is shown
                            autoFocus
                        />
                        <button
                            type="submit"
                            className="w-full bg-blue-600 text-white py-3 rounded-lg font-semibold text-lg hover:bg-blue-700 transition-colors shadow-lg disabled:bg-gray-400 disabled:cursor-not-allowed"
                            disabled={showCorrectAnswer} // Disable submit button while correct answer is shown
                        >
                            Kiểm tra & Kế tiếp
                        </button>
                    </form>
                </>
            )}
        </section>
    );
};


// --- Study Mode Component (Flashcard Display) ---
const StudyModeComponent = ({ vocabulary, topics, setModal }) => {
    const [filteredVocabulary, setFilteredVocabulary] = useState([]);
    const [currentCardIndex, setCurrentCardIndex] = useState(0);
    const [showVietnamese, setShowVietnamese] = useState(false);
    const [selectedTopic, setSelectedTopic] = useState(''); // Stores topic ID

    useEffect(() => {
        // Filter vocabulary based on selected topic
        let newFilteredVocab = vocabulary;
        if (selectedTopic) {
            newFilteredVocab = vocabulary.filter(voc => voc.topicId === selectedTopic);
        }
        setFilteredVocabulary(shuffleArray(newFilteredVocab)); // Shuffle on initial load or filter change
        setCurrentCardIndex(0);
        setShowVietnamese(false);
    }, [vocabulary, selectedTopic]);

    // Shuffle array function (re-defined or import from utility if needed)
    const shuffleArray = (array) => {
        let shuffled = [...array];
        for (let i = shuffled.length - 1; i > 0; i--) {
            const j = Math.floor(Math.random() * (i + 1));
            [shuffled[i], shuffled[j]] = [shuffled[j], shuffled[i]];
        }
        return shuffled;
    };

    const handleNextCard = () => {
        if (currentCardIndex < filteredVocabulary.length - 1) {
            setCurrentCardIndex(prev => prev + 1);
        } else {
            setModal({
                show: true,
                title: "Hoàn thành",
                message: "Bạn đã xem hết tất cả các flashcard trong chủ đề này!",
                onClose: () => setModal({ ...setModal, show: false })
            });
            setCurrentCardIndex(0); // Loop back to start or finish
        }
        setShowVietnamese(false);
    };

    const handlePreviousCard = () => {
        if (currentCardIndex > 0) {
            setCurrentCardIndex(prev => prev - 1);
        } else {
            setModal({
                show: true,
                title: "Thông báo",
                message: "Bạn đang ở flashcard đầu tiên.",
                onClose: () => setModal({ ...setModal, show: false })
            });
        }
        setShowVietnamese(false);
    };

    const currentCard = filteredVocabulary[currentCardIndex];

    if (filteredVocabulary.length === 0) {
        return (
            <section className="bg-white rounded-xl shadow-xl p-8 max-w-2xl mx-auto text-center">
                <h2 className="text-3xl font-bold text-blue-700 mb-6">Học nhanh từ vựng</h2>
                <p className="text-gray-700 mb-4">
                    Không có từ vựng nào để hiển thị. Vui lòng thêm từ vựng hoặc chọn chủ đề khác.
                </p>
                <div>
                    <label htmlFor="studyTopic" className="block text-gray-700 text-sm font-semibold mb-2">
                        Chọn chủ đề:
                    </label>
                    <select
                        id="studyTopic"
                        className="w-full px-4 py-2 border border-gray-300 rounded-lg bg-white focus:outline-none focus:ring-2 focus:ring-blue-500"
                        value={selectedTopic}
                        onChange={(e) => setSelectedTopic(e.target.value)}
                    >
                        <option value="">Tất cả chủ đề</option>
                        {topics.map(topic => (
                            <option key={topic.id} value={topic.id}>{topic.name}</option>
                        ))}
                    </select>
                </div>
            </section>
        );
    }

    return (
        <section className="bg-white rounded-xl shadow-xl p-8 max-w-2xl mx-auto text-center">
            <h2 className="text-3xl font-bold text-blue-700 mb-6">Học nhanh từ vựng</h2>
            <div className="mb-4">
                <label htmlFor="studyTopic" className="block text-gray-700 text-sm font-semibold mb-2">
                    Lọc theo chủ đề:
                </label>
                <select
                    id="studyTopic"
                    className="w-full px-4 py-2 border border-gray-300 rounded-lg bg-white focus:outline-none focus:ring-2 focus:ring-blue-500"
                    value={selectedTopic}
                    onChange={(e) => setSelectedTopic(e.target.value)}
                >
                    <option value="">Tất cả chủ đề</option>
                    {topics.map(topic => (
                        <option key={topic.id} value={topic.id}>{topic.name}</option>
                    ))}
                </select>
            </div>

            <div
                className="flashcard-content bg-gradient-to-br from-blue-500 to-blue-700 text-white p-10 rounded-xl shadow-xl cursor-pointer transform transition-transform duration-300 hover:scale-105 min-h-[200px] flex items-center justify-center relative overflow-hidden"
                onClick={() => setShowVietnamese(!showVietnamese)}
            >
                <div className="absolute top-4 left-4 text-sm opacity-80">
                    {currentCardIndex + 1} / {filteredVocabulary.length}
                </div>
                {!showVietnamese ? (
                    <p className="text-5xl font-extrabold text-center drop-shadow-lg break-words max-w-full">
                        {currentCard.english}
                    </p>
                ) : (
                    <p className="text-4xl font-semibold text-center drop-shadow-lg animate-fade-in-up break-words max-w-full">
                        {currentCard.vietnamese}
                    </p>
                )}
            </div>

            <div className="flex justify-center space-x-4 mt-8">
                <button
                    onClick={handlePreviousCard}
                    className="flex items-center bg-gray-200 text-gray-800 px-6 py-3 rounded-lg font-semibold hover:bg-gray-300 transition-colors shadow-md"
                >
                    <svg xmlns="http://www.w3.org/2000/svg" width="24" height="24" viewBox="0 0 24 24" fill="none" stroke="currentColor" strokeWidth="2" strokeLinecap="round" strokeLinejoin="round" className="lucide lucide-arrow-left mr-2"><path d="m12 19-7-7 7-7"/><path d="M19 12H5"/></svg>
                    Trước
                </button>
                <button
                    onClick={handleNextCard}
                    className="flex items-center bg-blue-600 text-white px-6 py-3 rounded-lg font-semibold hover:bg-blue-700 transition-colors shadow-md"
                >
                    Tiếp theo
                    <svg xmlns="http://www.w3.org/2000/svg" width="24" height="24" viewBox="0 0 24 24" fill="none" stroke="currentColor" strokeWidth="2" strokeLinecap="round" strokeLinejoin="round" className="lucide lucide-arrow-right ml-2"><path d="M5 12h14"/><path d="m12 5 7 7-7 7"/></svg>
                </button>
            </div>
        </section>
    );
};

// --- App Wrapper with Firebase Provider ---
const WrappedApp = () => (
    <FirebaseProvider>
        <App />
    </FirebaseProvider>
);

export default WrappedApp;
