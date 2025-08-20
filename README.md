import React, { useState, useEffect } from 'react';

// Firebase Imports
import { initializeApp } from 'firebase/app';
import { getAuth, signInAnonymously, signInWithCustomToken, onAuthStateChanged } from 'firebase/auth';
import { getFirestore, doc, setDoc, onSnapshot, collection, query, orderBy, getDocs, addDoc } from 'firebase/firestore';

// Lucide-React Icons (imported via a CDN-like approach)
// This is a common pattern to avoid local file imports in a single-file React app
const {
  Book, PlusCircle, Target, Sun, Moon, Check, X, ChevronRight, ChevronDown,
  ChevronLeft, LayoutDashboard, UploadCloud, ChevronUp, ChevronDown, Sparkles
} = window['lucide-react'];

// All component code and logic is contained within this single App component.
const App = () => {
  // === State Management ===
  const [db, setDb] = useState(null);
  const [auth, setAuth] = useState(null);
  const [userId, setUserId] = useState(null);
  const [isAuthReady, setIsAuthReady] = useState(false);
  const [currentPage, setCurrentPage] = useState('home');
  const [darkMode, setDarkMode] = useState(true);
  const [chapters, setChapters] = useState([]);
  const [isLoading, setIsLoading] = useState(false);
  const [isUploading, setIsUploading] = useState(false);
  const [practiceData, setPracticeData] = useState(null);
  const [practiceState, setPracticeState] = useState({
    currentQuestionIndex: 0,
    showAnswer: false,
    score: 0,
    attemptedQuestions: 0
  });

  // === Firebase Initialization & Authentication ===
  useEffect(() => {
    const initializeFirebase = async () => {
      try {
        // Global variables provided by the Canvas environment
        const appId = typeof __app_id !== 'undefined' ? __app_id : 'default-app-id';
        const firebaseConfig = JSON.parse(typeof __firebase_config !== 'undefined' ? __firebase_config : '{}');
        const initialAuthToken = typeof __initial_auth_token !== 'undefined' ? __initial_auth_token : null;

        // Initialize Firebase services
        const app = initializeApp(firebaseConfig);
        const firestore = getFirestore(app);
        const authService = getAuth(app);
        setDb(firestore);
        setAuth(authService);

        // Listen for authentication state changes
        const unsubscribe = onAuthStateChanged(authService, async (user) => {
          if (user) {
            setUserId(user.uid);
          } else {
            // Sign in anonymously if no user is found
            try {
              await signInAnonymously(authService);
            } catch (error) {
              console.error("Anonymous sign-in failed:", error);
            }
          }
          setIsAuthReady(true);
        });

        // Use custom token if available
        if (initialAuthToken) {
          try {
            await signInWithCustomToken(authService, initialAuthToken);
          } catch (error) {
            console.error("Custom token sign-in failed:", error);
          }
        }

        return () => unsubscribe();
      } catch (error) {
        console.error("Firebase initialization failed:", error);
      }
    };
    initializeFirebase();
  }, []);

  // === Firestore Data Listeners ===
  useEffect(() => {
    if (!isAuthReady || !db || !userId) return;

    // Path for private user data
    const appId = typeof __app_id !== 'undefined' ? __app_id : 'default-app-id';
    const chaptersPath = `artifacts/${appId}/users/${userId}/chapters`;
    const q = query(collection(db, chaptersPath));

    const unsubscribe = onSnapshot(q, (querySnapshot) => {
      const chaptersArray = [];
      querySnapshot.forEach((doc) => {
        chaptersArray.push({ id: doc.id, ...doc.data() });
      });
      setChapters(chaptersArray);
    }, (error) => {
      console.error("Error getting real-time chapter updates:", error);
    });

    // Clean up listener on component unmount
    return () => unsubscribe();
  }, [isAuthReady, db, userId]);

  // === Mock Functions ===
  const mockAIProcessChapter = async (file) => {
    setIsUploading(true);
    // Simulate API call delay
    await new Promise(resolve => setTimeout(resolve, 2000));
    // Mock the AI analysis result
    const mockContent = {
      title: file.name.replace(/\.pdf$/, ''),
      class: 'Class 10',
      subject: 'Science',
      topic: 'Biology',
      content: "This is the mocked text from your uploaded chapter. The AI would have extracted key concepts and definitions here.",
      keyConcepts: [
        { concept: 'Photosynthesis', description: 'The process used by plants to convert light energy into chemical energy.' },
        { concept: 'Cellular Respiration', description: 'The process of a cell converting sugar into energy.' },
      ],
      questionAreas: ['Photosynthesis reactants', 'Products of cellular respiration', 'Role of stomata'],
      progress: {
        completed: false,
        lastAttempt: null,
        averageScore: 0
      }
    };
    setIsUploading(false);
    return mockContent;
  };

  const mockAIGenerateQuestions = (chapter) => {
    // Simulate AI question generation
    const questions = [
      {
        type: 'MCQ',
        question: 'What are the main products of photosynthesis?',
        options: ['Water and Oxygen', 'Glucose and Water', 'Glucose and Oxygen', 'Carbon Dioxide and Water'],
        correctAnswer: 'Glucose and Oxygen',
        explanation: 'Photosynthesis uses light energy to convert carbon dioxide and water into glucose (a sugar) and oxygen.'
      },
      {
        type: 'Fill in the blanks',
        question: 'The process by which plants release water vapor is called _.',
        answer: 'Transpiration',
        explanation: 'Transpiration is the process of water movement through a plant and its evaporation from aerial parts, such as leaves, stems and flowers.'
      },
      {
        type: 'True/False',
        question: 'True or False: Cellular respiration occurs only in animal cells.',
        answer: 'False',
        explanation: 'Cellular respiration occurs in both plant and animal cells to produce energy.'
      },
      {
        type: 'Short answers',
        question: 'Briefly explain the role of chloroplasts in a plant cell.',
        answer: 'Chloroplasts are the sites of photosynthesis. They contain chlorophyll, a pigment that absorbs light energy to convert carbon dioxide and water into glucose and oxygen.',
        explanation: 'Chloroplasts are the organelles responsible for photosynthesis in plant cells. They contain chlorophyll which captures sunlight for the process.'
      },
      {
        type: 'MCQ',
        question: 'Which of the following is an input for cellular respiration?',
        options: ['Oxygen', 'Water', 'Carbon Dioxide', 'Sunlight'],
        correctAnswer: 'Oxygen',
        explanation: 'Cellular respiration requires oxygen and glucose to produce energy, carbon dioxide, and water.'
      },
      {
        type: 'Short answers',
        question: 'What is the role of stomata?',
        answer: 'Stomata are tiny pores on the surface of leaves that are responsible for gas exchange, allowing carbon dioxide to enter and oxygen to exit.',
        explanation: 'Stomata control the gas exchange and water transpiration in plants.'
      },
    ];
    return questions.slice(0, 5); // Just a few questions for the demo
  };

  // === Event Handlers ===
  const handleFileUpload = async (e) => {
    const file = e.target.files[0];
    if (!file) return;

    setIsLoading(true);
    try {
      const chapterData = await mockAIProcessChapter(file);

      // Save chapter data to Firestore
      const appId = typeof __app_id !== 'undefined' ? __app_id : 'default-app-id';
      const chaptersCollection = collection(db, `artifacts/${appId}/users/${userId}/chapters`);
      await addDoc(chaptersCollection, chapterData);

      setCurrentPage('library');
      setIsLoading(false);
    } catch (error) {
      console.error("Error processing or uploading chapter:", error);
      setIsLoading(false);
    }
  };

  const startPractice = (chapter) => {
    const questions = mockAIGenerateQuestions(chapter);
    setPracticeData({ chapter, questions });
    setPracticeState({ currentQuestionIndex: 0, showAnswer: false, score: 0, attemptedQuestions: 0 });
    setCurrentPage('practice');
  };

  const handleAnswer = (userAnswer) => {
    const currentQuestion = practiceData.questions[practiceState.currentQuestionIndex];
    const isCorrect = currentQuestion.type === 'True/False' || currentQuestion.type === 'MCQ'
      ? userAnswer === currentQuestion.correctAnswer
      : userAnswer.toLowerCase() === currentQuestion.answer.toLowerCase();

    const newScore = practiceState.score + (isCorrect ? 1 : 0);

    setPracticeState(prev => ({
      ...prev,
      score: newScore,
      attemptedQuestions: prev.attemptedQuestions + 1,
      isCorrect,
      showAnswer: true
    }));
  };

  const handleNextQuestion = () => {
    setPracticeState(prev => ({
      ...prev,
      currentQuestionIndex: prev.currentQuestionIndex + 1,
      showAnswer: false
    }));
  };

  // === Helper Functions for UI ===
  const getProgressStats = () => {
    const completedChapters = chapters.filter(c => c.progress.completed).length;
    const totalChapters = chapters.length;
    const totalScore = chapters.reduce((sum, c) => sum + c.progress.averageScore, 0);
    const averageScore = totalChapters > 0 ? (totalScore / totalChapters).toFixed(1) : 0;
    return { completedChapters, totalChapters, averageScore };
  };

  const getCategorizedChapters = () => {
    const categorized = {};
    chapters.forEach(c => {
      if (!categorized[c.class]) categorized[c.class] = {};
      if (!categorized[c.class][c.subject]) categorized[c.class][c.subject] = {};
      if (!categorized[c.class][c.subject][c.topic]) categorized[c.class][c.subject][c.topic] = [];
      categorized[c.class][c.subject][c.topic].push(c);
    });
    return categorized;
  };

  // === Reusable Components ===
  const NavButton = ({ text, icon, onClick, active }) => (
    <button
      onClick={onClick}
      className={`flex items-center space-x-2 px-4 py-3 rounded-xl transition-all duration-300
        ${active ? 'bg-purple-600 text-white shadow-lg' : 'text-gray-400 hover:bg-gray-700 hover:text-white'}`}
    >
      {icon}
      <span>{text}</span>
    </button>
  );

  // === Pages/Views ===
  const HomePage = () => (
    <div className="flex flex-col items-center justify-center min-h-screen-minus-header text-center p-4">
      <h1 className="text-5xl sm:text-6xl font-extrabold text-white mb-4 animate-fade-in">
        Your AI Study Companion
      </h1>
      <p className="text-lg text-gray-400 mb-8 max-w-2xl animate-fade-in delay-200">
        Empower your learning with smart, personalized practice and progress tracking.
      </p>
      <div className="flex flex-col sm:flex-row space-y-4 sm:space-y-0 sm:space-x-4 mb-8">
        <NavButton
          text="Add New Chapter"
          icon={<PlusCircle size={20} />}
          onClick={() => setCurrentPage('addChapter')}
        />
        <NavButton
          text="My Library"
          icon={<Book size={20} />}
          onClick={() => setCurrentPage('library')}
        />
        <NavButton
          text="Start Practice"
          icon={<Target size={20} />}
          onClick={() => {
            if (chapters.length > 0) {
              startPractice(chapters[0]); // Start practice with the first available chapter for demo
            } else {
              alert('Please add a chapter to start practicing.'); // In a real app, use a custom modal
            }
          }}
        />
      </div>
      <div className="absolute bottom-4 left-4 p-2 text-xs text-gray-500 rounded-lg bg-gray-800">
        User ID: <span className="font-mono text-gray-300 break-all">{userId || 'Loading...'}</span>
      </div>
    </div>
  );

  const AddChapterPage = () => (
    <div className="container mx-auto p-8 max-w-2xl">
      <h2 className="text-3xl font-bold text-white mb-6">Add New Chapter</h2>
      <p className="text-gray-400 mb-8">
        Upload a PDF to let the AI process it for you.
      </p>
      <div className="bg-gray-800 p-8 rounded-2xl shadow-xl flex flex-col items-center space-y-6">
        {isUploading ? (
          <div className="flex flex-col items-center space-y-4">
            <div className="animate-spin rounded-full h-16 w-16 border-t-2 border-b-2 border-purple-500"></div>
            <p className="text-gray-300 text-lg">Processing chapter with AI...</p>
          </div>
        ) : (
          <>
            <UploadCloud size={64} className="text-purple-500 mb-4" />
            <label htmlFor="file-upload" className="cursor-pointer bg-purple-600 text-white font-semibold py-3 px-8 rounded-full shadow-lg hover:bg-purple-700 transition-colors duration-300">
              Select PDF File
            </label>
            <input id="file-upload" type="file" accept=".pdf" onChange={handleFileUpload} className="hidden" />
          </>
        )}
      </div>
    </div>
  );

  const LibraryPage = () => {
    const categorizedChapters = getCategorizedChapters();
    const [openCategories, setOpenCategories] = useState({});

    const toggleCategory = (category) => {
      setOpenCategories(prev => ({
        ...prev,
        [category]: !prev[category]
      }));
    };

    return (
      <div className="container mx-auto p-8">
        <h2 className="text-3xl font-bold text-white mb-6">My Library</h2>
        {chapters.length === 0 ? (
          <div className="bg-gray-800 rounded-2xl p-8 text-center text-gray-400">
            <p>Your library is empty. Add a new chapter to get started!</p>
          </div>
        ) : (
          <div className="space-y-6">
            {Object.keys(categorizedChapters).map(classKey => (
              <div key={classKey} className="bg-gray-800 rounded-2xl p-6 shadow-xl">
                <button
                  onClick={() => toggleCategory(classKey)}
                  className="flex justify-between items-center w-full text-left text-xl font-semibold text-white mb-4"
                >
                  <span>{classKey}</span>
                  {openCategories[classKey] ? <ChevronUp /> : <ChevronDown />}
                </button>
                {openCategories[classKey] && (
                  <div className="space-y-4 ml-4">
                    {Object.keys(categorizedChapters[classKey]).map(subjectKey => (
                      <div key={subjectKey}>
                        <h4 className="text-lg font-medium text-gray-300 flex items-center mb-2">
                          <ChevronRight size={16} className="text-purple-400 mr-1" />{subjectKey}
                        </h4>
                        <ul className="list-disc list-inside ml-6 space-y-2">
                          {Object.keys(categorizedChapters[classKey][subjectKey]).map(topicKey => (
                            <li key={topicKey}>
                              <span className="text-gray-400 font-bold">{topicKey}:</span>
                              <ul className="ml-4 space-y-1">
                                {categorizedChapters[classKey][subjectKey][topicKey].map(chapter => (
                                  <li key={chapter.id} className="text-gray-400 flex justify-between items-center">
                                    <span>{chapter.title}</span>
                                    <button
                                      onClick={() => startPractice(chapter)}
                                      className="bg-purple-600 text-white text-xs px-4 py-1 rounded-full hover:bg-purple-700 transition-colors duration-300"
                                    >
                                      Start Practice
                                    </button>
                                  </li>
                                ))}
                              </ul>
                            </li>
                          ))}
                        </ul>
                      </div>
                    ))}
                  </div>
                )}
              </div>
            ))}
          </div>
        )}
      </div>
    );
  };

  const PracticePage = () => {
    if (!practiceData) return <div className="p-8 text-center text-gray-400">No practice session started.</div>;

    const { chapter, questions } = practiceData;
    const { currentQuestionIndex, showAnswer, score, attemptedQuestions } = practiceState;
    const currentQuestion = questions[currentQuestionIndex];
    const isLastQuestion = currentQuestionIndex >= questions.length - 1;

    return (
      <div className="container mx-auto p-8 max-w-3xl">
        <h2 className="text-3xl font-bold text-white mb-4">Practice: {chapter.title}</h2>
        <div className="bg-gray-800 p-8 rounded-2xl shadow-xl space-y-6">
          <p className="text-gray-400 text-sm">
            Question {currentQuestionIndex + 1} of {questions.length}
          </p>
          <div className="bg-gray-700 p-6 rounded-lg">
            <h3 className="text-lg font-semibold text-white mb-4">{currentQuestion.question}</h3>

            {/* Render different question types */}
            {!showAnswer && (
              <>
                {currentQuestion.type === 'MCQ' && (
                  <div className="space-y-2">
                    {currentQuestion.options.map((option, index) => (
                      <button
                        key={index}
                        onClick={() => handleAnswer(option)}
                        className="w-full text-left bg-gray-600 text-white p-3 rounded-lg hover:bg-purple-600 transition-colors duration-200"
                      >
                        {option}
                      </button>
                    ))}
                  </div>
                )}
                {(currentQuestion.type === 'Fill in the blanks' || currentQuestion.type === 'Short answers' || currentQuestion.type === 'Long answers') && (
                  <input
                    type="text"
                    placeholder="Type your answer here..."
                    onKeyPress={(e) => {
                      if (e.key === 'Enter') handleAnswer(e.target.value);
                    }}
                    className="w-full bg-gray-600 text-white p-3 rounded-lg border-2 border-transparent focus:border-purple-500 focus:outline-none transition-colors duration-200"
                  />
                )}
                {currentQuestion.type === 'True/False' && (
                  <div className="flex space-x-4">
                    <button onClick={() => handleAnswer('True')} className="flex-1 bg-gray-600 text-white p-3 rounded-lg hover:bg-purple-600">True</button>
                    <button onClick={() => handleAnswer('False')} className="flex-1 bg-gray-600 text-white p-3 rounded-lg hover:bg-purple-600">False</button>
                  </div>
                )}
              </>
            )}

            {showAnswer && (
              <div className="mt-4 p-4 rounded-lg border-2" style={{
                borderColor: practiceState.isCorrect ? '#4CAF50' : '#F44336',
                backgroundColor: practiceState.isCorrect ? 'rgba(76, 175, 80, 0.1)' : 'rgba(244, 67, 54, 0.1)'
              }}>
                <div className="flex items-center space-x-2 mb-2">
                  {practiceState.isCorrect ? (
                    <Check size={24} className="text-green-500" />
                  ) : (
                    <X size={24} className="text-red-500" />
                  )}
                  <p className={`font-semibold ${practiceState.isCorrect ? 'text-green-500' : 'text-red-500'}`}>
                    {practiceState.isCorrect ? 'Correct!' : 'Incorrect'}
                  </p>
                </div>
                <p className="text-sm text-gray-300">
                  <span className="font-semibold text-white">Explanation:</span> {currentQuestion.explanation}
                </p>
              </div>
            )}

            {showAnswer && (
              <div className="mt-6 flex justify-end">
                <button
                  onClick={handleNextQuestion}
                  className="bg-purple-600 text-white px-6 py-2 rounded-full font-semibold hover:bg-purple-700 transition-colors"
                >
                  {isLastQuestion ? 'Finish Practice' : 'Next Question'}
                </button>
              </div>
            )}
          </div>
        </div>
      </div>
    );
  };

  const ProgressTrackerPage = () => {
    const { completedChapters, totalChapters, averageScore } = getProgressStats();
    return (
      <div className="container mx-auto p-8 max-w-3xl">
        <h2 className="text-3xl font-bold text-white mb-6">Progress Tracker</h2>
        <div className="bg-gray-800 rounded-2xl shadow-xl p-8 space-y-6">
          <div className="grid grid-cols-1 sm:grid-cols-3 gap-6 text-center">
            <div className="bg-gray-700 p-6 rounded-xl">
              <p className="text-purple-400 text-4xl font-bold">{completedChapters}</p>
              <p className="text-gray-400 text-sm mt-2">Chapters Completed</p>
            </div>
            <div className="bg-gray-700 p-6 rounded-xl">
              <p className="text-purple-400 text-4xl font-bold">{averageScore}</p>
              <p className="text-gray-400 text-sm mt-2">Average Score</p>
            </div>
            <div className="bg-gray-700 p-6 rounded-xl">
              <p className="text-purple-400 text-4xl font-bold">0</p>
              <p className="text-gray-400 text-sm mt-2">Daily Quiz Streak</p>
            </div>
          </div>
          <div className="bg-gray-700 p-6 rounded-xl">
            <h3 className="text-xl font-semibold text-white mb-2">Weak Areas</h3>
            <ul className="text-gray-400 list-disc list-inside">
              <li>Photosynthesis Concepts</li>
              <li>Mughal Empire Dates</li>
            </ul>
          </div>
        </div>
      </div>
    );
  };

  const renderPage = () => {
    switch (currentPage) {
      case 'home':
        return <HomePage />;
      case 'addChapter':
        return <AddChapterPage />;
      case 'library':
        return <LibraryPage />;
      case 'practice':
        return <PracticePage />;
      case 'tracker':
        return <ProgressTrackerPage />;
      default:
        return <HomePage />;
    }
  };

  // Main App Component Structure
  return (
    <div className={`min-h-screen ${darkMode ? 'bg-gray-900 text-gray-200' : 'bg-gray-50 text-gray-800'} transition-colors duration-300`}>
      {/* Header and Navigation */}
      <header className="fixed top-0 left-0 w-full z-50 bg-gray-900/80 backdrop-blur-md shadow-lg">
        <nav className="container mx-auto p-4 flex justify-between items-center">
          <button onClick={() => setCurrentPage('home')} className="text-xl font-bold text-purple-400">
            Study AI
          </button>
          <div className="flex space-x-2">
            <button onClick={() => setCurrentPage('library')} className="p-2 text-gray-400 hover:text-white transition-colors duration-200 rounded-full">
              <Book size={20} />
            </button>
            <button onClick={() => setCurrentPage('tracker')} className="p-2 text-gray-400 hover:text-white transition-colors duration-200 rounded-full">
              <LayoutDashboard size={20} />
            </button>
            <button onClick={() => setDarkMode(!darkMode)} className="p-2 text-gray-400 hover:text-white transition-colors duration-200 rounded-full">
              {darkMode ? <Sun size={20} /> : <Moon size={20} />}
            </button>
          </div>
        </nav>
      </header>
      
      {/* Main Content Area */}
      <main className="pt-20">
        {renderPage()}
      </main>
      
      {/* Loading Overlay */}
      {isLoading && (
        <div className="fixed inset-0 bg-black bg-opacity-70 flex items-center justify-center z-50">
          <div className="flex flex-col items-center space-y-4">
            <div className="animate-spin rounded-full h-16 w-16 border-t-2 border-b-2 border-purple-500"></div>
            <p className="text-white text-lg font-semibold">Loading...</p>
          </div>
        </div>
      )}
    </div>
  );
};

export default App;
