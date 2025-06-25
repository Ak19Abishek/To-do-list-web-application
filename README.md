import React, { useState, useEffect, createContext, useContext } from 'react';
import { initializeApp } from 'firebase/app';
import { getAuth, signInAnonymously, signInWithCustomToken, onAuthStateChanged, signOut } from 'firebase/auth';
import { getFirestore, collection, addDoc, onSnapshot, doc, updateDoc, deleteDoc, query, orderBy, serverTimestamp } from 'firebase/firestore';

// --- Firebase Context ---
const FirebaseContext = createContext(null);

// Firebase Provider component to initialize Firebase and provide auth/db to children
function FirebaseProvider({ children }) {
    const [app, setApp] = useState(null);
    const [auth, setAuth] = useState(null);
    const [db, setDb] = useState(null);
    const [userId, setUserId] = useState(null);
    const [isAuthReady, setIsAuthReady] = useState(false);

    useEffect(() => {
        // Initialize Firebase app and services
        try {
            const firebaseConfig = typeof __firebase_config !== 'undefined' ? JSON.parse(__firebase_config) : {};
            const initializedApp = initializeApp(firebaseConfig);
            const initializedAuth = getAuth(initializedApp);
            const initializedDb = getFirestore(initializedApp);

            setApp(initializedApp);
            setAuth(initializedAuth);
            setDb(initializedDb);

            // Handle initial authentication state and subsequent changes
            const unsubscribe = onAuthStateChanged(initializedAuth, async (user) => {
                if (user) {
                    setUserId(user.uid);
                    setIsAuthReady(true);
                } else {
                    // Sign in with custom token if available, otherwise anonymously
                    const initialAuthToken = typeof __initial_auth_token !== 'undefined' ? __initial_auth_token : null;
                    try {
                        if (initialAuthToken) {
                            await signInWithCustomToken(initializedAuth, initialAuthToken);
                        } else {
                            await signInAnonymously(initializedAuth);
                        }
                    } catch (error) {
                        console.error("Firebase Auth Error:", error);
                        // Fallback or error message to user
                    }
                }
            });

            return () => unsubscribe(); // Cleanup auth listener
        } catch (error) {
            console.error("Failed to initialize Firebase:", error);
        }
    }, []); // Run only once on mount

    return (
        <FirebaseContext.Provider value={{ app, auth, db, userId, isAuthReady }}>
            {children}
        </FirebaseContext.Provider>
    );
}

// Custom hook to use Firebase services
function useFirebase() {
    return useContext(FirebaseContext);
}

// --- Components ---

// Modal component for confirmations
const Modal = ({ show, title, message, onConfirm, onCancel }) => {
    if (!show) return null;

    return (
        <div className="fixed inset-0 bg-gray-600 bg-opacity-50 flex items-center justify-center p-4 z-50">
            <div className="bg-white p-6 rounded-lg shadow-xl max-w-sm w-full border-4 border-indigo-500 transform transition-all duration-300 scale-100 opacity-100">
                <h3 className="text-xl font-bold text-gray-800 mb-4">{title}</h3>
                <p className="text-gray-700 mb-6">{message}</p>
                <div className="flex justify-end space-x-3">
                    <button
                        onClick={onCancel}
                        className="px-5 py-2 bg-gray-300 text-gray-800 rounded-lg hover:bg-gray-400 transition duration-200 ease-in-out font-semibold shadow-md focus:outline-none focus:ring-2 focus:ring-gray-500 focus:ring-opacity-75"
                    >
                        Cancel
                    </button>
                    <button
                        onClick={onConfirm}
                        className="px-5 py-2 bg-red-600 text-white rounded-lg hover:bg-red-700 transition duration-200 ease-in-out font-semibold shadow-md focus:outline-none focus:ring-2 focus:ring-red-500 focus:ring-opacity-75"
                    >
                        Confirm
                    </button>
                </div>
            </div>
        </div>
    );
};


// TodoForm component for adding/editing tasks
const TodoForm = ({ currentTodo, onSave, onCancelEdit }) => {
    const [title, setTitle] = useState(currentTodo?.title || '');
    const [description, setDescription] = useState(currentTodo?.description || '');
    const [deadline, setDeadline] = useState(currentTodo?.deadline ? new Date(currentTodo.deadline.toDate()).toISOString().slice(0, 16) : '');

    useEffect(() => {
        if (currentTodo) {
            setTitle(currentTodo.title);
            setDescription(currentTodo.description || '');
            setDeadline(currentTodo.deadline ? new Date(currentTodo.deadline.toDate()).toISOString().slice(0, 16) : '');
        } else {
            setTitle('');
            setDescription('');
            setDeadline('');
        }
    }, [currentTodo]);

    const handleSubmit = (e) => {
        e.preventDefault();
        if (!title.trim()) return;

        const todoData = {
            title,
            description,
            deadline: deadline ? new Date(deadline) : null,
            isCompleted: currentTodo?.isCompleted || false,
        };
        onSave(todoData);
        setTitle('');
        setDescription('');
        setDeadline('');
    };

    return (
        <form onSubmit={handleSubmit} className="bg-white p-6 rounded-xl shadow-lg border border-gray-200 mb-8 max-w-2xl mx-auto">
            <h2 className="text-3xl font-extrabold text-gray-800 mb-6 text-center">{currentTodo ? 'Edit Task' : 'Add New Task'}</h2>
            <div className="mb-4">
                <label htmlFor="title" className="block text-gray-700 text-sm font-bold mb-2">Task Title</label>
                <input
                    type="text"
                    id="title"
                    value={title}
                    onChange={(e) => setTitle(e.target.value)}
                    placeholder="e.g., Buy groceries"
                    className="shadow appearance-none border rounded-lg w-full py-3 px-4 text-gray-700 leading-tight focus:outline-none focus:ring-2 focus:ring-indigo-500 focus:border-transparent transition duration-200 ease-in-out"
                    required
                />
            </div>
            <div className="mb-4">
                <label htmlFor="description" className="block text-gray-700 text-sm font-bold mb-2">Description (Optional)</label>
                <textarea
                    id="description"
                    value={description}
                    onChange={(e) => setDescription(e.target.value)}
                    placeholder="Details about the task..."
                    rows="3"
                    className="shadow appearance-none border rounded-lg w-full py-3 px-4 text-gray-700 leading-tight focus:outline-none focus:ring-2 focus:ring-indigo-500 focus:border-transparent transition duration-200 ease-in-out resize-y"
                ></textarea>
            </div>
            <div className="mb-6">
                <label htmlFor="deadline" className="block text-gray-700 text-sm font-bold mb-2">Deadline (Optional)</label>
                <input
                    type="datetime-local"
                    id="deadline"
                    value={deadline}
                    onChange={(e) => setDeadline(e.target.value)}
                    className="shadow appearance-none border rounded-lg w-full py-3 px-4 text-gray-700 leading-tight focus:outline-none focus:ring-2 focus:ring-indigo-500 focus:border-transparent transition duration-200 ease-in-out"
                />
            </div>
            <div className="flex justify-end space-x-4">
                {currentTodo && (
                    <button
                        type="button"
                        onClick={onCancelEdit}
                        className="bg-gray-300 hover:bg-gray-400 text-gray-800 font-bold py-3 px-6 rounded-lg transition duration-200 ease-in-out shadow-md focus:outline-none focus:ring-2 focus:ring-gray-500 focus:ring-opacity-75"
                    >
                        Cancel
                    </button>
                )}
                <button
                    type="submit"
                    className="bg-indigo-600 hover:bg-indigo-700 text-white font-bold py-3 px-6 rounded-lg transition duration-200 ease-in-out shadow-lg transform hover:scale-105 focus:outline-none focus:ring-2 focus:ring-indigo-500 focus:ring-opacity-75"
                >
                    {currentTodo ? 'Update Task' : 'Add Task'}
                </button>
            </div>
        </form>
    );
};


// TodoItem component for displaying individual todo
const TodoItem = ({ todo, onToggleComplete, onEdit, onDelete }) => {
    const now = new Date();
    const deadlineDate = todo.deadline ? new Date(todo.deadline.toDate()) : null;

    const isOverdue = deadlineDate && deadlineDate < now && !todo.isCompleted;
    const isDueSoon = deadlineDate && deadlineDate > now && (deadlineDate.getTime() - now.getTime() < 24 * 60 * 60 * 1000) && !todo.isCompleted; // Within 24 hours

    const itemClasses = `
        bg-white p-5 rounded-xl shadow-md border
        flex items-center justify-between transition-all duration-300 ease-in-out
        ${todo.isCompleted ? 'opacity-60 bg-green-50 border-green-300' : ''}
        ${isOverdue ? 'bg-red-50 border-red-400' : ''}
        ${isDueSoon ? 'bg-yellow-50 border-yellow-300' : ''}
    `;

    const titleClasses = `
        font-semibold text-lg
        ${todo.isCompleted ? 'line-through text-gray-500' : 'text-gray-800'}
    `;

    const formatDate = (date) => {
        if (!date) return '';
        return new Date(date.toDate()).toLocaleString('en-US', {
            year: 'numeric', month: 'short', day: 'numeric',
            hour: '2-digit', minute: '2-digit', hour12: true
        });
    };

    return (
        <div className={itemClasses}>
            <div className="flex items-center flex-grow">
                <input
                    type="checkbox"
                    checked={todo.isCompleted}
                    onChange={() => onToggleComplete(todo.id, !todo.isCompleted)}
                    className="form-checkbox h-6 w-6 text-indigo-600 rounded-md transition duration-150 ease-in-out cursor-pointer flex-shrink-0"
                />
                <div className="ml-4 flex-grow">
                    <h3 className={titleClasses}>{todo.title}</h3>
                    {todo.description && (
                        <p className="text-sm text-gray-600 mt-1">{todo.description}</p>
                    )}
                    {todo.deadline && (
                        <p className="text-xs text-gray-500 mt-2 flex items-center">
                            <svg className="w-4 h-4 mr-1 text-indigo-500" fill="currentColor" viewBox="0 0 20 20" xmlns="http://www.w3.org/2000/svg"><path fillRule="evenodd" d="M6 2a1 1 0 00-1 1v1H4a2 2 0 00-2 2v10a2 2 0 002 2h12a2 2 0 002-2V6a2 2 0 00-2-2h-1V3a1 1 0 10-2 0v1H7V3a1 1 0 00-1-1zm0 5a1 1 0 000 2h8a1 1 0 100-2H6z" clipRule="evenodd"></path></svg>
                            Deadline: {formatDate(todo.deadline)}
                            {isOverdue && <span className="ml-2 text-red-600 font-bold">(Overdue!)</span>}
                            {isDueSoon && <span className="ml-2 text-yellow-600 font-bold">(Due Soon!)</span>}
                        </p>
                    )}
                </div>
            </div>
            <div className="flex space-x-2 ml-4 flex-shrink-0">
                <button
                    onClick={() => onEdit(todo)}
                    className="p-2 rounded-full bg-blue-100 text-blue-600 hover:bg-blue-200 transition duration-150 ease-in-out focus:outline-none focus:ring-2 focus:ring-blue-500 focus:ring-opacity-75"
                    title="Edit Task"
                >
                    <svg className="w-5 h-5" fill="currentColor" viewBox="0 0 20 20" xmlns="http://www.w3.org/2000/svg"><path d="M13.586 3.586a2 2 0 112.828 2.828l-.793.793-2.828-2.828.793-.793zm-3.586 3.586l-5 5V15h5l5-5-2.828-2.828z"></path></svg>
                </button>
                <button
                    onClick={() => onDelete(todo.id)}
                    className="p-2 rounded-full bg-red-100 text-red-600 hover:bg-red-200 transition duration-150 ease-in-out focus:outline-none focus:ring-2 focus:ring-red-500 focus:ring-opacity-75"
                    title="Delete Task"
                >
                    <svg className="w-5 h-5" fill="currentColor" viewBox="0 0 20 20" xmlns="http://www.w3.org/2000/svg"><path fillRule="evenodd" d="M9 2a1 1 0 00-.894.553L7.382 4H4a1 1 0 000 2v10a2 2 0 002 2h8a2 2 0 002-2V6a1 1 0 100-2h-3.382l-.724-1.447A1 1 0 0011 2H9zM7 8a1 1 0 012 0v6a1 1 0 11-2 0V8zm6 0a1 1 0 01-2 0v6a1 1 0 112 0V8z" clipRule="evenodd"></path></svg>
                </button>
            </div>
        </div>
    );
};

// Main App component
function App() {
    const { db, userId, isAuthReady } = useFirebase();
    const [todos, setTodos] = useState([]);
    const [currentTodo, setCurrentTodo] = useState(null); // For editing
    const [showDeleteModal, setShowDeleteModal] = useState(false);
    const [todoToDelete, setTodoToDelete] = useState(null);

    // Fetch todos when auth is ready and userId exists
    useEffect(() => {
        if (!isAuthReady || !db || !userId) {
            console.log("Firebase not ready or no user ID.");
            setTodos([]); // Clear todos if not authenticated or ready
            return;
        }

        const appId = typeof __app_id !== 'undefined' ? __app_id : 'default-app-id';
        const todosCollectionRef = collection(db, `artifacts/${appId}/users/${userId}/todos`);
        const q = query(todosCollectionRef, orderBy('createdAt', 'desc')); // Order by creation time

        const unsubscribe = onSnapshot(q, (snapshot) => {
            const todosData = snapshot.docs.map(doc => ({
                id: doc.id,
                ...doc.data()
            }));
            setTodos(todosData);
        }, (error) => {
            console.error("Error fetching todos: ", error);
        });

        return () => unsubscribe(); // Cleanup listener on unmount or userId/db change
    }, [isAuthReady, db, userId]);

    // Add or Update Todo
    const handleSaveTodo = async (todoData) => {
        if (!db || !userId) return;
        const appId = typeof __app_id !== 'undefined' ? __app_id : 'default-app-id';
        const todosCollectionRef = collection(db, `artifacts/${appId}/users/${userId}/todos`);

        try {
            if (currentTodo) {
                // Update existing todo
                const todoRef = doc(db, todosCollectionRef, currentTodo.id);
                await updateDoc(todoRef, {
                    ...todoData,
                    deadline: todoData.deadline ? serverTimestamp(todoData.deadline) : null,
                });
                setCurrentTodo(null); // Exit edit mode
                console.log("Todo updated successfully!");
            } else {
                // Add new todo
                await addDoc(todosCollectionRef, {
                    ...todoData,
                    createdAt: serverTimestamp(),
                    deadline: todoData.deadline ? serverTimestamp(todoData.deadline) : null,
                });
                console.log("Todo added successfully!");
            }
        } catch (error) {
            console.error("Error saving todo:", error);
        }
    };

    // Toggle Todo Completion Status
    const handleToggleComplete = async (id, isCompleted) => {
        if (!db || !userId) return;
        const appId = typeof __app_id !== 'undefined' ? __app_id : 'default-app-id';
        const todoRef = doc(db, `artifacts/${appId}/users/${userId}/todos`, id);
        try {
            await updateDoc(todoRef, { isCompleted: isCompleted });
            console.log("Todo completion toggled!");
        } catch (error) {
            console.error("Error toggling todo completion:", error);
        }
    };

    // Prepare for Delete
    const handleDeleteClick = (id) => {
        setTodoToDelete(id);
        setShowDeleteModal(true);
    };

    // Confirm Delete
    const handleConfirmDelete = async () => {
        if (!db || !userId || !todoToDelete) return;
        const appId = typeof __app_id !== 'undefined' ? __app_id : 'default-app-id';
        const todoRef = doc(db, `artifacts/${appId}/users/${userId}/todos`, todoToDelete);
        try {
            await deleteDoc(todoRef);
            console.log("Todo deleted successfully!");
        } catch (error) {
            console.error("Error deleting todo:", error);
        } finally {
            setShowDeleteModal(false);
            setTodoToDelete(null);
        }
    };

    // Cancel Delete
    const handleCancelDelete = () => {
        setShowDeleteModal(false);
        setTodoToDelete(null);
    };

    return (
        <div className="min-h-screen bg-gradient-to-br from-indigo-50 to-purple-100 font-inter text-gray-900 py-10 px-4 sm:px-6 lg:px-8">
            <style>
                {`
                @import url('https://fonts.googleapis.com/css2?family=Inter:wght@300;400;600;700;800&display=swap');
                body { font-family: 'Inter', sans-serif; }
                /* Custom scrollbar for better aesthetics */
                ::-webkit-scrollbar {
                    width: 8px;
                }
                ::-webkit-scrollbar-track {
                    background: #f1f1f1;
                    border-radius: 10px;
                }
                ::-webkit-scrollbar-thumb {
                    background: #888;
                    border-radius: 10px;
                }
                ::-webkit-scrollbar-thumb:hover {
                    background: #555;
                }
                `}
            </style>
            <div className="max-w-4xl mx-auto">
                <header className="text-center mb-10">
                    <h1 className="text-5xl font-extrabold text-indigo-700 leading-tight">
                        My Productive To-Do List
                    </h1>
                    <p className="mt-3 text-lg text-gray-600">
                        Organize your tasks, set deadlines, and achieve your goals.
                    </p>
                    {userId && (
                        <p className="mt-4 text-sm text-gray-500 bg-white p-3 rounded-lg shadow-inner inline-block">
                            Logged in as: <span className="font-mono text-gray-700 break-all">{userId}</span>
                        </p>
                    )}
                </header>

                <TodoForm
                    currentTodo={currentTodo}
                    onSave={handleSaveTodo}
                    onCancelEdit={() => setCurrentTodo(null)}
                />

                <section className="mt-10">
                    <h2 className="text-3xl font-extrabold text-gray-800 mb-6 text-center">Your Tasks</h2>
                    {todos.length === 0 && isAuthReady && (
                        <p className="text-center text-gray-500 text-lg mt-8 p-6 bg-white rounded-xl shadow-md border border-gray-200">
                            You don't have any tasks yet. Add one above!
                        </p>
                    )}
                    <div className="space-y-4">
                        {todos.map(todo => (
                            <TodoItem
                                key={todo.id}
                                todo={todo}
                                onToggleComplete={handleToggleComplete}
                                onEdit={setCurrentTodo} // Set currentTodo for editing
                                onDelete={handleDeleteClick} // Use the modal trigger
                            />
                        ))}
                    </div>
                </section>
            </div>

            <Modal
                show={showDeleteModal}
                title="Confirm Deletion"
                message="Are you sure you want to delete this task? This action cannot be undone."
                onConfirm={handleConfirmDelete}
                onCancel={handleCancelDelete}
            />
        </div>
    );
}

// Ensure the App component is exported as default for Canvas
export default function AppWrapper() {
    return (
        <FirebaseProvider>
            <App />
        </FirebaseProvider>
    );
}
