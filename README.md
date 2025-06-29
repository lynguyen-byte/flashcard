import React, { useState, useEffect, useRef, useCallback } from 'react';
import { initializeApp } from 'firebase/app';
import { getAuth, signInAnonymously, signInWithCustomToken, onAuthStateChanged } from 'firebase/auth';
import { getFirestore, collection, addDoc, doc, deleteDoc, onSnapshot, query, where } from 'firebase/firestore';

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
    const [isTeacherMode, setIsTeacherMode] = useState(false); // Client-side flag for teacher demo

    // Initialize Firebase and authenticate user
    useEffect(() => {
        const initFirebase = async () => {
            try {
                const app = initializeApp(firebaseConfig);
                const authInstance = getAuth(app);
                const dbInstance = getFirestore(app);

                setAuth(authInstance);
                setDb(dbInstance);

                if (initialAuthToken) {
                    await signInWithCustomToken(authInstance, initialAuthToken);
                } else {
                    await signInAnonymously(authInstance);
                }

                const unsubscribe = onAuthStateChanged(authInstance, (user) => {
                    if (user) {
                        setUserId(user.uid);
                    } else {
                        setUserId(crypto.randomUUID()); // Anonymous ID
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
                <div className="text-sm text-gray-600 mb-4 flex items-center space-x-4">
                    <span>ID người dùng của bạn: <span className="font-mono bg-gray-200 px-2 py-1 rounded">{userId}</span></span>
                    <label className="inline-flex items-center ml-4">
                        <input
                            type="checkbox"
                            className="form-checkbox h-5 w-5 text-blue-600"
                            checked={isTeacherMode}
                            onChange={(e) => setIsTeacherMode(e.target.checked)}
                        />
                        <span className="ml-2 text-blue-700 font-semibold">Tôi là Giáo viên (Demo)</span>
                    </label>
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
                    <ManageVocabulary db={db} auth={auth} userId={userId} showMessage={showMessage} isTeacherMode={isTeacherMode} />
                )}
                {(currentView === 'review' || currentView === 'test' || currentView === 'flipcard') && db && auth && userId && (
                    <ReviewSection db={db} auth={auth} userId={userId} showMessage={showMessage} setCurrentView={setCurrentView} currentView={currentView} />
                )}
            </div>
        </div>
    );
};

// Component to manage vocabulary (add, import, list)
const ManageVocabulary = ({ db, auth, userId, showMessage, isTeacherMode }) => {
    const [privateLessons, setPrivateLessons] = useState([]);
    const [publicLessons, setPublicLessons] = useState([]);
    const [allFlashcards, setAllFlashcards] = useState([]); // Store all flashcards (private and public)
    const [newLessonName, setNewLessonName] = useState('');
    const [isNewLessonPublic, setIsNewLessonPublic] = useState(false);
    const [englishWord, setEnglishWord] = useState('');
    const [vietnameseWord, setVietnameseWord] = useState('');
    const [selectedLessonId, setSelectedLessonId] = useState(''); // Can be private or public lesson ID
    const [bulkText, setBulkText] = useState('');
    const [bulkLessonId, setBulkLessonId] = useState(''); // Can be private or public lesson ID
    const [ocrLoading, setOcrLoading] = useState(false);

    // Fetch lessons and flashcards
    useEffect(() => {
        if (!db || !userId) return;

        // Listener for private lessons
        const privateLessonsQuery = query(collection(db, `artifacts/${appId}/users/${userId}/lessons`));
        const unsubscribePrivateLessons = onSnapshot(privateLessonsQuery, (snapshot) => {
            const lessonsData = snapshot.docs.map(doc => ({ id: doc.id, ...doc.data(), isPublic: false }));
            setPrivateLessons(lessonsData);
        }, (error) => {
            console.error("Error loading private lessons:", error);
            showMessage('Không thể tải bài học cá nhân.', 'error');
        });

        // Listener for public lessons
        const publicLessonsQuery = query(collection(db, `artifacts/${appId}/shared_lessons`));
        const unsubscribePublicLessons = onSnapshot(publicLessonsQuery, (snapshot) => {
            const lessonsData = snapshot.docs.map(doc => ({ id: doc.id, ...doc.data(), isPublic: true }));
            setPublicLessons(lessonsData);
        }, (error) => {
            console.error("Error loading public lessons:", error);
            showMessage('Không thể tải bài học công khai.', 'error');
        });

        // Listener for private flashcards
        const privateFlashcardsQuery = query(collection(db, `artifacts/${appId}/users/${userId}/flashcards`));
        const unsubscribePrivateFlashcards = onSnapshot(privateFlashcardsQuery, (snapshot) => {
            const cardsData = snapshot.docs.map(doc => ({ id: doc.id, ...doc.data(), isPublic: false }));
            // Merge with public flashcards for display
            setAllFlashcards(prev => {
                const publicCards = prev.filter(card => card.isPublic);
                return [...publicCards, ...cardsData];
            });
        }, (error) => {
            console.error("Error loading private flashcards:", error);
            showMessage('Không thể tải flashcard cá nhân.', 'error');
        });

        // Listener for public flashcards
        const publicFlashcardsQuery = query(collection(db, `artifacts/${appId}/shared_flashcards`));
        const unsubscribePublicFlashcards = onSnapshot(publicFlashcardsQuery, (snapshot) => {
            const cardsData = snapshot.docs.map(doc => ({ id: doc.id, ...doc.data(), isPublic: true }));
            // Merge with private flashcards for display
            setAllFlashcards(prev => {
                const privateCards = prev.filter(card => !card.isPublic && card.ownerId === userId); // Ensure private cards are correctly filtered by ownerId
                return [...privateCards, ...cardsData];
            });
        }, (error) => {
            console.error("Error loading public flashcards:", error);
            showMessage('Không thể tải flashcard công khai.', 'error');
        });

        return () => {
            unsubscribePrivateLessons();
            unsubscribePublicLessons();
            unsubscribePrivateFlashcards();
            unsubscribePublicFlashcards();
        };
    }, [db, userId, showMessage]);

    // Set default selected lesson if available and not already set
    useEffect(() => {
        if (privateLessons.length > 0 && !selectedLessonId) {
            setSelectedLessonId(privateLessons[0].id);
            setBulkLessonId(privateLessons[0].id);
        } else if (publicLessons.length > 0 && !selectedLessonId && privateLessons.length === 0) {
             setSelectedLessonId(publicLessons[0].id);
             setBulkLessonId(publicLessons[0].id);
        }
    }, [privateLessons, publicLessons, selectedLessonId]);


    // Helper to determine if a lesson is public based on its ID (prefixed for clarity)
    const getLessonType = (lessonId) => {
        const lesson = [...privateLessons, ...publicLessons].find(l => l.id === lessonId);
        return lesson ? lesson.isPublic : false;
    };

    // Handle adding a new lesson
    const handleAddLesson = async () => {
        if (!newLessonName.trim() || !db || !userId) {
            showMessage('Tên bài học không được trống.', 'error');
            return;
        }

        const lessonCollectionPath = isNewLessonPublic ? `artifacts/${appId}/shared_lessons` : `artifacts/${appId}/users/${userId}/lessons`;
        try {
            await addDoc(collection(db, lessonCollectionPath), {
                name: newLessonName.trim(),
                ownerId: userId, // Store owner ID for public lessons
                createdAt: new Date(),
            });
            setNewLessonName('');
            setIsNewLessonPublic(false);
            showMessage('Thêm bài học thành công!', 'success');
        } catch (e) {
            console.error("Lỗi khi thêm bài học:", e);
            showMessage('Lỗi khi thêm bài học.', 'error');
        }
    };

    // Handle adding a single flashcard
    const handleAddFlashcard = async () => {
        if (!englishWord.trim() || !vietnameseWord.trim() || !selectedLessonId || !db || !userId) {
            showMessage('Vui lòng điền đầy đủ từ vựng và chọn bài học.', 'error');
            return;
        }

        const isPublicCard = getLessonType(selectedLessonId);
        if (isPublicCard && !isTeacherMode) {
             showMessage('Bạn không có quyền thêm flashcard vào bài học công khai.', 'error');
             return;
        }
        const flashcardCollectionPath = isPublicCard ? `artifacts/${appId}/shared_flashcards` : `artifacts/${appId}/users/${userId}/flashcards`;

        try {
            await addDoc(collection(db, flashcardCollectionPath), {
                englishWord: englishWord.trim(),
                vietnameseWord: vietnameseWord.trim(),
                lessonId: selectedLessonId, // Reference to the lesson (private or public)
                ownerId: userId, // Store owner ID for public flashcards
                isPublic: isPublicCard,
                createdAt: new Date(),
            });
            setEnglishWord('');
            setVietnameseWord('');
            showMessage('Thêm flashcard thành công!', 'success');
        } catch (e) {
            console.error("Lỗi khi thêm flashcard:", e);
            showMessage('Lỗi khi thêm flashcard.', 'error');
        }
    };

    // Handle bulk import of flashcards
    const handleBulkImport = async () => {
        if (!bulkText.trim() || !bulkLessonId || !db || !userId) {
            showMessage('Vui lòng nhập văn bản và chọn bài học.', 'error');
            return;
        }

        const isPublicBulkCard = getLessonType(bulkLessonId);
        if (isPublicBulkCard && !isTeacherMode) {
             showMessage('Bạn không có quyền thêm flashcard vào bài học công khai.', 'error');
             return;
        }
        const flashcardCollectionPath = isPublicBulkCard ? `artifacts/${appId}/shared_flashcards` : `artifacts/${appId}/users/${userId}/flashcards`;

        const lines = bulkText.split('\n').filter(line => line.trim() !== '');
        const newFlashcards = [];
        for (const line of lines) {
            const parts = line.split('-').map(part => part.trim());
            if (parts.length >= 2) {
                newFlashcards.push({
                    englishWord: parts[0],
                    vietnameseWord: parts.slice(1).join(' - '),
                    lessonId: bulkLessonId,
                    ownerId: userId,
                    isPublic: isPublicBulkCard,
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
                addDoc(collection(db, flashcardCollectionPath), card)
            ));
            setBulkText('');
            showMessage(`Đã thêm ${newFlashcards.length} flashcard thành công!`, 'success');
        } catch (e) {
            console.error("Lỗi khi nhập hàng loạt:", e);
            showMessage('Lỗi khi nhập hàng loạt flashcard.', 'error');
        }
    };

    // Handle image upload and OCR
    const handleImageUploadAndOCR = async (e) => {
        const file = e.target.files[0];
        if (!file) return;

        setOcrLoading(true);
        showMessage('Đang phân tích hình ảnh, vui lòng đợi...', 'info');

        const reader = new FileReader();
        reader.onloadend = async () => {
            const base64ImageData = reader.result.split(',')[1];

            try {
                const prompt = "Extract all English words and their Vietnamese meanings from this image. Present them as 'English Word - Vietnamese Meaning' on separate lines. If no Vietnamese meaning is available, just provide the English word. Ignore any other text not related to vocabulary pairs.";
                const payload = {
                    contents: [
                        {
                            role: "user",
                            parts: [{ text: prompt }, { inlineData: { mimeType: file.type, data: base64ImageData } }]
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
                if (result.candidates && result.candidates.length > 0 && result.candidates[0].content && result.candidates[0].content.parts && result.candidates[0].content.parts.length > 0) {
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
    const handleDeleteLesson = async (lessonId, isPublicLesson) => {
        if (!db || !userId) return;

        if (isPublicLesson && !isTeacherMode) {
            showMessage('Bạn không có quyền xóa bài học công khai.', 'error');
            return;
        }
        const collectionPath = isPublicLesson ? `artifacts/${appId}/shared_lessons` : `artifacts/${appId}/users/${userId}/lessons`;
        const flashcardCollectionPath = isPublicLesson ? `artifacts/${appId}/shared_flashcards` : `artifacts/${appId}/users/${userId}/flashcards`;

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
            await deleteDoc(doc(db, collectionPath, lessonId));

            // Delete associated flashcards
            const cardsToDelete = allFlashcards.filter(card => card.lessonId === lessonId && card.isPublic === isPublicLesson);
            await Promise.all(cardsToDelete.map(card =>
                deleteDoc(doc(db, flashcardCollectionPath, card.id))
            ));
            showMessage('Xóa bài học và các flashcard liên quan thành công!', 'success');
        } catch (error) {
            console.error("Lỗi khi xóa bài học:", error);
            showMessage('Lỗi khi xóa bài học.', 'error');
        }
    };

    // Delete a single flashcard
    const handleDeleteFlashcard = async (cardId, isPublicCard) => {
        if (!db || !userId) return;

        if (isPublicCard && !isTeacherMode) {
            showMessage('Bạn không có quyền xóa flashcard công khai.', 'error');
            return;
        }

        const flashcardCollectionPath = isPublicCard ? `artifacts/${appId}/shared_flashcards` : `artifacts/${appId}/users/${userId}/flashcards`;

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
            await deleteDoc(doc(db, flashcardCollectionPath, cardId));
            showMessage('Xóa flashcard thành công!', 'success');
        } catch (error) {
            console.error("Lỗi khi xóa flashcard:", error);
            showMessage('Lỗi khi xóa flashcard.', 'error');
        }
    };


    const allAvailableLessons = [...privateLessons, ...publicLessons];


    return (
        <div className="space-y-8">
            {/* Thêm bài học mới */}
            <div className="bg-blue-50 p-6 rounded-lg shadow-inner">
                <h2 className="text-2xl font-semibold text-blue-700 mb-4">Thêm Bài học mới</h2>
                <div className="flex flex-col sm:flex-row gap-3 items-center">
                    <input
                        type="text"
                        placeholder="Tên bài học (VD: Bài 1, Từ vựng IELTS...)"
                        value={newLessonName}
                        onChange={(e) => setNewLessonName(e.target.value)}
                        className="flex-grow p-3 border border-blue-300 rounded-md focus:ring-2 focus:ring-blue-400 outline-none"
                    />
                    {isTeacherMode && (
                        <label className="inline-flex items-center text-blue-700 font-semibold cursor-pointer whitespace-nowrap">
                            <input
                                type="checkbox"
                                className="form-checkbox h-5 w-5 text-purple-600 rounded mr-2"
                                checked={isNewLessonPublic}
                                onChange={(e) => setIsNewLessonPublic(e.target.checked)}
                            />
                            Tạo công khai
                        </label>
                    )}
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
                        {privateLessons.map(lesson => (
                            <option key={lesson.id} value={lesson.id}>{lesson.name} (Cá nhân)</option>
                        ))}
                        {publicLessons.map(lesson => (
                            <option key={lesson.id} value={lesson.id} disabled={!isTeacherMode} className={!isTeacherMode ? 'opacity-50 cursor-not-allowed' : ''}>
                                {lesson.name} (Công khai - {lesson.ownerId === userId ? 'Của bạn' : 'Khác'})
                            </option>
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
                     {privateLessons.map(lesson => (
                        <option key={lesson.id} value={lesson.id}>{lesson.name} (Cá nhân)</option>
                    ))}
                    {publicLessons.map(lesson => (
                        <option key={lesson.id} value={lesson.id} disabled={!isTeacherMode} className={!isTeacherMode ? 'opacity-50 cursor-not-allowed' : ''}>
                            {lesson.name} (Công khai - {lesson.ownerId === userId ? 'Của bạn' : 'Khác'})
                        </option>
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
                {(privateLessons.length === 0 && publicLessons.length === 0) ? (
                    <p className="text-gray-500">Chưa có bài học nào. Hãy thêm một bài học mới!</p>
                ) : (
                    <div className="space-y-6">
                        {/* Private Lessons */}
                        {privateLessons.length > 0 && (
                            <div className="border border-gray-200 rounded-lg p-4 bg-white shadow-sm">
                                <h3 className="text-xl font-medium text-gray-800 mb-3">Bài học Cá nhân của bạn</h3>
                                {privateLessons.map(lesson => (
                                    <div key={lesson.id} className="border-b border-gray-100 last:border-b-0 py-2">
                                        <div className="flex justify-between items-center mb-1">
                                            <span className="font-semibold text-gray-700">
                                                {lesson.name} ({allFlashcards.filter(card => !card.isPublic && card.lessonId === lesson.id).length} từ)
                                            </span>
                                            <button
                                                onClick={() => handleDeleteLesson(lesson.id, false)}
                                                className="text-red-500 hover:text-red-700 transition-colors"
                                                title="Xóa bài học và tất cả từ vựng"
                                            >
                                                <svg xmlns="http://www.w3.org/2000/svg" className="h-5 w-5" fill="none" viewBox="0 0 24 24" stroke="currentColor">
                                                    <path strokeLinecap="round" strokeLinejoin="round" strokeWidth={2} d="M19 7l-.867 12.142A2 2 0 0116.138 21H7.862a2 2 0 01-1.995-1.858L5 7m5 4v6m4-6v6m1-10V4a1 1 0 00-1-1h-4a1 1 0 00-1 1v3M4 7h16" />
                                                </svg>
                                            </button>
                                        </div>
                                        {allFlashcards.filter(card => !card.isPublic && card.lessonId === lesson.id).length > 0 ? (
                                            <ul className="text-sm ml-4">
                                                {allFlashcards
                                                    .filter(card => !card.isPublic && card.lessonId === lesson.id)
                                                    .map(card => (
                                                        <li key={card.id} className="flex justify-between items-center py-1">
                                                            <span className="text-gray-600">
                                                                {card.englishWord} - {card.vietnameseWord}
                                                            </span>
                                                            <button
                                                                onClick={() => handleDeleteFlashcard(card.id, false)}
                                                                className="text-red-400 hover:text-red-600 text-xs"
                                                                title="Xóa flashcard"
                                                            >
                                                                Xóa
                                                            </button>
                                                        </li>
                                                    ))}
                                            </ul>
                                        ) : (
                                            <p className="text-gray-500 text-xs ml-4">Chưa có từ vựng nào trong bài học này.</p>
                                        )}
                                    </div>
                                ))}
                            </div>
                        )}

                        {/* Public Lessons */}
                        {publicLessons.length > 0 && (
                            <div className="border border-gray-200 rounded-lg p-4 bg-white shadow-sm">
                                <h3 className="text-xl font-medium text-gray-800 mb-3">Bài học Công khai</h3>
                                {publicLessons.map(lesson => (
                                    <div key={lesson.id} className="border-b border-gray-100 last:border-b-0 py-2">
                                        <div className="flex justify-between items-center mb-1">
                                            <span className="font-semibold text-gray-700">
                                                {lesson.name} ({allFlashcards.filter(card => card.isPublic && card.lessonId === lesson.id).length} từ)
                                                {lesson.ownerId === userId ? " (Của bạn)" : ` (Tạo bởi: ${lesson.ownerId.substring(0, 8)}...)`}
                                            </span>
                                            {isTeacherMode && lesson.ownerId === userId && ( // Only allow teacher to delete their own public lesson
                                                <button
                                                    onClick={() => handleDeleteLesson(lesson.id, true)}
                                                    className="text-red-500 hover:text-red-700 transition-colors"
                                                    title="Xóa bài học công khai của bạn"
                                                >
                                                    <svg xmlns="http://www.w3.org/2000/svg" className="h-5 w-5" fill="none" viewBox="0 0 24 24" stroke="currentColor">
                                                        <path strokeLinecap="round" strokeLinejoin="round" strokeWidth={2} d="M19 7l-.867 12.142A2 2 0 0116.138 21H7.862a2 2 0 01-1.995-1.858L5 7m5 4v6m4-6v6m1-10V4a1 1 0 00-1-1h-4a1 1 0 00-1 1v3M4 7h16" />
                                                    </svg>
                                                </button>
                                            )}
                                        </div>
                                        {allFlashcards.filter(card => card.isPublic && card.lessonId === lesson.id).length > 0 ? (
                                            <ul className="text-sm ml-4">
                                                {allFlashcards
                                                    .filter(card => card.isPublic && card.lessonId === lesson.id)
                                                    .map(card => (
                                                        <li key={card.id} className="flex justify-between items-center py-1">
                                                            <span className="text-gray-600">
                                                                {card.englishWord} - {card.vietnameseWord}
                                                            </span>
                                                            {isTeacherMode && card.ownerId === userId && ( // Only allow teacher to delete their own public flashcard
                                                                <button
                                                                    onClick={() => handleDeleteFlashcard(card.id, true)}
                                                                    className="text-red-400 hover:text-red-600 text-xs"
                                                                    title="Xóa flashcard công khai của bạn"
                                                                >
                                                                    Xóa
                                                                </button>
                                                            )}
                                                        </li>
                                                    ))}
                                            </ul>
                                        ) : (
                                            <p className="text-gray-500 text-xs ml-4">Chưa có từ vựng nào trong bài học này.</p>
                                        )}
                                    </div>
                                ))}
                            </div>
                        )}
                    </div>
                )}
            </div>
        </div>
    );
};

// Component for Review Configuration and starting tests
const ReviewSection = ({ db, auth, userId, showMessage, setCurrentView, currentView }) => {
    const [privateLessons, setPrivateLessons] = useState([]);
    const [publicLessons, setPublicLessons] = useState([]);
    const [allFlashcards, setAllFlashcards] = useState([]);
    const [selectedPrivateLessonIds, setSelectedPrivateLessonIds] = useState([]);
    const [selectedPublicLessonIds, setSelectedPublicLessonIds] = useState([]);
    const [numQuestions, setNumQuestions] = useState(10);
    const [testDuration, setTestDuration] = useState(60); // in seconds, for test mode
    const [testMode, setTestMode] = useState('flip-card'); // 'eng-vie', 'vie-eng', 'flip-card'
    const [testFlashcards, setTestFlashcards] = useState([]); // Cards selected for the current test/flip session
    const [lastTestFlashcards, setLastTestFlashcards] = useState([]); // Stores cards from the last full test for re-testing

    // Fetch lessons and flashcards for review configuration
    useEffect(() => {
        if (!db || !userId) return;

        // Listener for private lessons
        const privateLessonsQuery = query(collection(db, `artifacts/${appId}/users/${userId}/lessons`));
        const unsubscribePrivateLessons = onSnapshot(privateLessonsQuery, (snapshot) => {
            const lessonsData = snapshot.docs.map(doc => ({ id: doc.id, ...doc.data(), type: 'private' }));
            setPrivateLessons(lessonsData);
        }, (error) => {
            console.error("Error loading private lessons for review:", error);
            showMessage('Không thể tải bài học cá nhân cho phần ôn tập.', 'error');
        });

        // Listener for public lessons
        const publicLessonsQuery = query(collection(db, `artifacts/${appId}/shared_lessons`));
        const unsubscribePublicLessons = onSnapshot(publicLessonsQuery, (snapshot) => {
            const lessonsD
