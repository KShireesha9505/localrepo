app.py

from flask import Flask, render_template, request, jsonify
from github2 import GitHubIssueResolver
from main3 import initialize_vector_store, format_issue_response, generate_solution
from dotenv import load_dotenv
import os
from langchain_google_genai import ChatGoogleGenerativeAI
from langchain.agents import AgentExecutor, create_react_agent
from langchain.tools.retriever import create_retriever_tool
from langchain import hub
from langchain_community.utilities import SerpAPIWrapper
from langchain.tools import Tool

load_dotenv()

app = Flask(__name__)

# Initialize components
resolver = GitHubIssueResolver()
vstore = initialize_vector_store()
threshold = 0.6
retriever = vstore.as_retriever(
    search_type="similarity_score_threshold",
    search_kwargs={
        "k": 3,
        "score_threshold": threshold
    }
)

# Configure agent tools
retriever_tool = create_retriever_tool(
    retriever,
    name="github_issues",
    description="Search existing GitHub issues and solutions"
)

llm = ChatGoogleGenerativeAI(
    model="gemini-1.5-pro",
    google_api_key=os.getenv("GEMINI_API_KEY")
)

def setup_web_search_tool():
    # """Configure web search tool"""
    # search = SerpAPIWrapper()
    # return Tool(
    #     name="web_search",
    #     func=search.run,
    #     descr
    # iption="Search the internet for technical solutions. Input should be a clear search query."
    # )
    """Configure SerpAPI for reliable web searches and extract top 3 URLs."""
    search = SerpAPIWrapper()

    def search_with_links(query):
        # Use the .results property to get structured results
        search_results = search.results(query)
        formatted_results = []

        # Get up to 3 results from 'organic_results'
        for result in search_results.get('organic_results', [])[:3]:
            title = result.get('title')
            snippet = result.get('snippet')
            link = result.get('link')

            if snippet and link:
                formatted_results.append(f"{title}\n{snippet}\nURL: {link}")

        return "\n\n".join(formatted_results) if formatted_results else "No results found."

    return Tool(
        name="web_search",
        func=search_with_links,
        description="Search the internet for technical solutions. Returns top 3 results with title, snippet, and URL."
    )


web_search_tool = setup_web_search_tool()
tools = [ generate_solution, web_search_tool]
agent = create_react_agent(llm, tools, hub.pull("hwchase17/react"))
executor = AgentExecutor(agent=agent, tools=tools, verbose=True)

@app.route('/')
def index():
    """Render the main interface"""
    return render_template('index.html')

@app.route('/api/issues', methods=['POST'])
def search_issues():
    """Handle issue search requests"""
    try:
        data = request.get_json()
        query = data.get('query', '').strip()
        
        if not query:
            return jsonify({"error": "Please enter a search query"}), 400
        
        # Search existing issues
        similar_docs = retriever.invoke(query)
        
        if similar_docs:
            resolution_status = resolver.get_issue_state(similar_docs[0].metadata['number'])
            formatted_response = format_issue_response(
                similar_docs,
                {
                    "status": "resolved" if resolution_status["is_resolved"] else "open",
                    "reference": similar_docs[0].metadata['url']
                }
            )
            return jsonify({
                "results": formatted_response,
                "has_results": True,
                "issues": [
                    {
                        "number": doc.metadata['number'],
                        "title": doc.metadata['title'],
                        "url": doc.metadata['url'],
                        "state": doc.metadata['state'],
                        "created_at": doc.metadata['created_at'],
                        "description": doc.page_content,
                        "solution": doc.metadata.get('solution', '')
                    } for doc in similar_docs
                ]
            })
        else:
            return jsonify({
                "results": "No similar issues found.",
                "has_results": False
            })
            
    except Exception as e:
        return jsonify({"error": str(e)}), 500

@app.route('/api/comment', methods=['POST'])
def add_comment():
    """Add comment to an existing issue"""
    try:
        data = request.get_json()
        issue_number = data.get('issue_number')
        comment = data.get('comment', '').strip()
        
        if not comment:
            return jsonify({"error": "Comment cannot be empty"}), 400
            
        msg = resolver.contribute_to_issue(issue_number, comment)
        return jsonify({"message": msg})
        
    except Exception as e:
        return jsonify({"error": str(e)}), 500

@app.route('/api/solution', methods=['POST'])
def get_solution():
    """Get AI-generated solution and relevant links"""
    try:
        data = request.get_json()
        problem = data.get('problem', '').strip()
        
        if not problem:
            return jsonify({"error": "Problem description cannot be empty"}), 400
        
        # Perform web search for relevant links
        print("Performing web search...")
        web_search_result = web_search_tool.func(problem)
        print("Web search result:", web_search_result)
        
        # Generate a consolidated solution
        print("Generating AI solution...")
        ai_solution = executor.invoke({
            "input": f"Search for technical solutions to: {problem}. Focus on StackOverflow and programming blogs."
        })
        print("AI solution result:", ai_solution)
        
        # Return both the web search results and the AI solution
        return jsonify({
            "links": web_search_result,
            "solution": ai_solution["output"]
        })
        
    except Exception as e:
        print("Error in /api/solution:", str(e))  # Debugging
        return jsonify({"error": str(e)}), 500

@app.route('/api/update-db', methods=['POST'])
def update_database():
    """Update the issues database"""
    global vstore, retriever
    
    try:
        issues = resolver.fetch_issues_as_documents()
        try:
            vstore.delete_collection()
        except:
            pass
        
        # Reinitialize the vector store
        vstore = initialize_vector_store()
        
        # Enhance issues with solutions from PRs/comments
        enhanced_issues = []
        for issue in issues:
            issue_number = issue.metadata['number']
            resolution_data = resolver.get_issue_state(issue_number)
            
            if resolution_data['is_resolved']:
                solution = f"Fixed in PR: {resolution_data.get('pr_url', 'N/A')}"
                issue.metadata['solution'] = solution
            enhanced_issues.append(issue)
        
        vstore.add_documents(enhanced_issues)
        retriever = vstore.as_retriever(
            search_type="similarity_score_threshold",
            search_kwargs={
                "k": 3,
                "score_threshold": threshold
            }
        )
        
        return jsonify({
            "message": f"Updated database with {len(enhanced_issues)} issues",
            "success": True
        })
        
    except Exception as e:
        return jsonify({
            "error": str(e),
            "success": False
        }), 500
@app.route('/results')
def results():
    query = request.args.get('query', '')
    return render_template('results.html', query=query)
if __name__ == '__main__':
    app.run(debug=True)


# from flask import Flask, render_template, request, jsonify
# from github2 import GitHubIssueResolver
# from main3 import initialize_vector_store, format_issue_response, generate_solution
# from dotenv import load_dotenv
# import os
# from langchain_google_genai import ChatGoogleGenerativeAI
# from langchain.agents import AgentExecutor, create_react_agent
# from langchain.tools.retriever import create_retriever_tool
# from langchain import hub
# from langchain_community.utilities import SerpAPIWrapper
# from langchain.tools import Tool

# load_dotenv()

# app = Flask(__name__)

# # Initialize components
# resolver = GitHubIssueResolver()
# vstore = initialize_vector_store()
# threshold = 0.6
# retriever = vstore.as_retriever(
#     search_type="similarity_score_threshold",
#     search_kwargs={
#         "k": 3,
#         "score_threshold": threshold
#     }
# )

# retriever_tool = create_retriever_tool(
#     retriever,
#     name="github_issues",
#     description="Search existing GitHub issues and solutions"
# )

# llm = ChatGoogleGenerativeAI(
#     model="gemini-1.5-pro",
#     google_api_key=os.getenv("GEMINI_API_KEY")
# )

# def setup_web_search_tool():
#     search = SerpAPIWrapper()
#     return Tool(
#         name="web_search",
#         func=search.run,
#         description="Search the internet for technical solutions. Input should be a clear search query."
#     )

# web_search_tool = setup_web_search_tool()
# tools = [retriever_tool, generate_solution, web_search_tool]
# agent = create_react_agent(llm, tools, hub.pull("hwchase17/react"))
# executor = AgentExecutor(agent=agent, tools=tools, verbose=True)

# @app.route('/')
# def index():
#     return render_template('index.html')

# @app.route('/results', methods=['POST'])
# def results_page():
#     """Show results page with similar issues and AI solution"""
#     try:
#         query = request.form.get('query', '').strip()
        
#         if not query:
#             return render_template('results.html', error="Please enter an issue to search.")
        
#         similar_docs = retriever.invoke(query)
#         ai_solution = None
#         formatted_response = None

#         if similar_docs:
#             resolution_status = resolver.get_issue_state(similar_docs[0].metadata['number'])
#             formatted_response = format_issue_response(
#                 similar_docs,
#                 {
#                     "status": "resolved" if resolution_status["is_resolved"] else "open",
#                     "reference": similar_docs[0].metadata['url']
#                 }
#             )
#         else:
#             formatted_response = "No similar issues found."

#         # Optional: Also get AI solution
#         ai_result = executor.invoke({
#             "input": f"Search for technical solutions to: {query}. Focus on StackOverflow and programming blogs."
#         })
#         ai_solution = ai_result["output"]

#         return render_template(
#             'results.html',
#             query=query,
#             issues=similar_docs,
#             results=formatted_response,
#             solution=ai_solution
#         )
    
#     except Exception as e:
#         return render_template('results.html', error=f"An error occurred: {str(e)}")

# @app.route('/api/comment', methods=['POST'])
# def add_comment():
#     try:
#         data = request.get_json()
#         issue_number = data.get('issue_number')
#         comment = data.get('comment', '').strip()
        
#         if not comment:
#             return jsonify({"error": "Comment cannot be empty"}), 400
            
#         msg = resolver.contribute_to_issue(issue_number, comment)
#         return jsonify({"message": msg})
        
#     except Exception as e:
#         return jsonify({"error": str(e)}), 500

# @app.route('/api/solution', methods=['POST'])
# def get_solution():
#     try:
#         data = request.get_json()
#         problem = data.get('problem', '').strip()
        
#         if not problem:
#             return jsonify({"error": "Problem description cannot be empty"}), 400
            
#         result = executor.invoke({
#             "input": f"Search for technical solutions to: {problem}. Focus on StackOverflow and programming blogs."
#         })
        
#         return jsonify({"solution": result["output"]})
        
#     except Exception as e:
#         return jsonify({"error": str(e)}), 500

# @app.route('/api/update-db', methods=['POST'])
# def update_database():
#     global vstore, retriever
    
#     try:
#         issues = resolver.fetch_issues_as_documents()
#         try:
#             vstore.delete_collection()
#         except:
#             pass
        
#         vstore = initialize_vector_store()
        
#         enhanced_issues = []
#         for issue in issues:
#             issue_number = issue.metadata['number']
#             resolution_data = resolver.get_issue_state(issue_number)
            
#             if resolution_data['is_resolved']:
#                 solution = f"Fixed in PR: {resolution_data.get('pr_url', 'N/A')}"
#                 issue.metadata['solution'] = solution
#             enhanced_issues.append(issue)
        
#         vstore.add_documents(enhanced_issues)
#         retriever = vstore.as_retriever(
#             search_type="similarity_score_threshold",
#             search_kwargs={
#                 "k": 3,
#                 "score_threshold": threshold
#             }
#         )
        
#         return jsonify({
#             "message": f"Updated database with {len(enhanced_issues)} issues",
#             "success": True
#         })
        
#     except Exception as e:
#         return jsonify({
#             "error": str(e),
#             "success": False
#         }), 500

# if __name__ == '__main__':
#     app.run(debug=True)
-------------------------------------------------------

index.html
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>GitHub Issue Resolver</title>
    <link rel="stylesheet" href="{{ url_for('static', filename='css/style.css') }}">
    <link href="https://fonts.googleapis.com/css2?family=Roboto:wght@300;400;500;700&display=swap" rel="stylesheet">
    <link rel="stylesheet" href="https://cdnjs.cloudflare.com/ajax/libs/font-awesome/6.0.0/css/all.min.css">
</head>
<body>
    <div class="container">
        <!-- Header Section -->
        <header class="welcome-header">
            <div class="welcome-banner">
                <h1><i class="fab fa-github"></i> GitHub Issue Resolver</h1>
                <p class="subtitle">AI-powered solutions for technical issues</p>
            </div>
        </header>

        <!-- Main Content -->
        <main>
            <!-- Features Section -->
            <section class="features-section">
                <div class="feature-card">
                    <div class="feature-icon">
                        <i class="fas fa-search"></i>
                    </div>
                    <h3>Search</h3>
                    <p>Find existing GitHub issues similar to yours</p>
                </div>
                
                <div class="feature-card">
                    <div class="feature-icon">
                        <i class="fas fa-check-circle"></i>
                    </div>
                    <h3>Solutions</h3>
                    <p>View community-approved fixes and workarounds</p>
                </div>
                
                <div class="feature-card">
                    <div class="feature-icon">
                        <i class="fas fa-comment-dots"></i>
                    </div>
                    <h3>Resolve</h3>
                    <p>Comment on issues or continue as new issues</p>
                </div>
                
                <div class="feature-card">
                    <div class="feature-icon">
                        <i class="fas fa-robot"></i>
                    </div>
                    <h3>AI Help</h3>
                    <p>Get automated diagnosis and prevention advice</p>
                </div>
            </section>

            <!-- Configuration Section -->
            <section class="config-section">
                <h2><i class="fas fa-cog"></i> Current Configuration</h2>
                <div class="config-grid">
                    <div class="config-item">
                        <span class="config-label">Repository:</span>
                        <span class="config-value">techwithtim/Flask-Web-App-Tutorial</span>
                    </div>
                    <div class="config-item">
                        <span class="config-label">AI Model:</span>
                        <span class="config-value">Gemini 1.5 Pro</span>
                    </div>
                    <div class="config-item">
                        <span class="config-label">Database:</span>
                        <span class="config-value">AstraDB</span>
                    </div>
                </div>
            </section>

            <!-- Search and Database Section -->
            <section class="search-section">
                <div class="search-card">
                    <h2><i class="fas fa-bug"></i> Describe Your Issue</h2>
                    <div class="search-box">
                        <textarea id="issue-query" placeholder="Enter details about the technical issue you're facing..."></textarea>
                        <button id="search-btn" class="primary-btn">
                            <i class="fas fa-search"></i> Search GitHub Issues
                        </button>
                    </div>
                </div>

                <div class="database-card">
                    <h2><i class="fas fa-database"></i> Database Management</h2>
                    <p>Update the local issue database with the latest from GitHub</p>
                    <button id="update-db-btn" class="secondary-btn">
                        <i class="fas fa-sync-alt"></i> Update Database Now
                    </button>
                    <div class="db-status">
                        <i class="fas fa-info-circle"></i> Last updated: <span id="last-updated">Loading...</span>
                    </div>
                </div>

                <div class="action-buttons">
                    <button id="ai-help-btn" class="accent-btn" style="display:none;">
                        <i class="fas fa-robot"></i> Get AI Help
                    </button>
                </div>
            </section>
        </main>
    </div>

    <script src="{{ url_for('static', filename='js/main.js') }}"></script>
</body>
</html>
---------------------------------------------------------------------
style.css

/* General Styles */
:root {
  --primary-color: #2c3e50;
  --secondary-color: #3498db;
  --accent-color: #e74c3c;
  --light-color: #ecf0f1;
  --dark-color: #2c3e50;
  --success-color: #2ecc71;
  --warning-color: #f39c12;
  --info-color: #3498db;
}

body {
  font-family: 'Roboto', sans-serif;
  background-color: #f5f7fa;
  color: #333;
  line-height: 1.6;
  margin: 0;
  padding: 0;
}

.container {
  max-width: 960px;
  margin: 0 auto;
  padding: 20px;
}

/* Header Styles */
.welcome-header {
  text-align: center;
  margin-bottom: 40px;
  background: linear-gradient(135deg, var(--primary-color), var(--secondary-color));
  color: white;
  padding: 30px 20px;
  border-radius: 10px;
  box-shadow: 0 4px 6px rgba(0, 0, 0, 0.1);
}

.welcome-header h1 {
  margin: 0;
  font-size: 2.5rem;
}

.subtitle {
  font-size: 1.2rem;
  opacity: 0.9;
  margin-top: 10px;
}

/* Features Section */
.features-section {
  display: grid;
  grid-template-columns: repeat(auto-fit, minmax(220px, 1fr));
  gap: 20px;
  margin-bottom: 40px;
}

.feature-card {
  background: white;
  padding: 20px;
  border-radius: 8px;
  text-align: center;
  box-shadow: 0 2px 4px rgba(0,0,0,0.05);
}

.feature-icon {
  font-size: 2rem;
  color: var(--secondary-color);
  margin-bottom: 10px;
}

/* Configuration Section */
.config-section {
  margin-bottom: 40px;
  background: white;
  padding: 25px;
  border-radius: 8px;
  box-shadow: 0 2px 4px rgba(0,0,0,0.05);
}

.config-section h2 {
  margin-top: 0;
  color: var(--primary-color);
  display: flex;
  align-items: center;
  gap: 10px;
}

.config-grid {
  display: grid;
  grid-template-columns: 1fr;
  gap: 10px;
  margin-top: 15px;
}

.config-item {
  display: flex;
  justify-content: space-between;
  padding: 10px;
  background-color: var(--light-color);
  border-radius: 4px;
}

.config-label {
  font-weight: 500;
  color: var(--primary-color);
}

/* Search Section */
.search-section {
  display: grid;
  gap: 30px;
}

.search-card, .database-card {
  background: white;
  padding: 25px;
  border-radius: 8px;
  box-shadow: 0 2px 4px rgba(0, 0, 0, 0.05);
}

.search-card h2, .database-card h2 {
  margin-top: 0;
  color: var(--primary-color);
  display: flex;
  align-items: center;
  gap: 10px;
}

.search-box {
  display: flex;
  flex-direction: column;
  gap: 15px;
}

.search-box textarea,
.search-box input[type="text"] {
  width: 100%;
  padding: 15px;
  border: 1px solid #ddd;
  border-radius: 4px;
  font-size: 1rem;
  font-family: 'Roboto', sans-serif;
  resize: vertical;
  box-sizing: border-box;
}

.database-card p {
  margin-bottom: 20px;
}

.db-status {
  margin-top: 15px;
  font-size: 0.9rem;
  color: #666;
  display: flex;
  align-items: center;
  gap: 5px;
}

/* Button Styles */
button {
  cursor: pointer;
  border: none;
  border-radius: 4px;
  padding: 12px 20px;
  font-size: 1rem;
  font-weight: 500;
  transition: all 0.3s ease;
  display: inline-flex;
  align-items: center;
  gap: 8px;
}

.primary-btn {
  background-color: var(--secondary-color);
  color: white;
}

.primary-btn:hover {
  background-color: #2980b9;
}

.secondary-btn {
  background-color: var(--light-color);
  color: var(--dark-color);
}

.secondary-btn:hover {
  background-color: #d5dbdb;
}

.accent-btn {
  background-color: var(--accent-color);
  color: white;
}

.accent-btn:hover {
  background-color: #c0392b;
}

/* Responsive Design */
@media (max-width: 768px) {
  .container {
    padding: 15px;
  }

  .welcome-header h1 {
    font-size: 2rem;
  }

  .search-box textarea,
  .search-box input[type="text"] {
    min-height: 100px;
  }

  .features-section {
    grid-template-columns: 1fr;
  }

  .config-grid {
    grid-template-columns: 1fr;
  }
}
---------------------------------------------------------------------

main.js

document.addEventListener('DOMContentLoaded', function() {
    // Load last updated time
    loadLastUpdated();
    
    // Search button functionality
    document.getElementById('search-btn').addEventListener('click', function() {
        const query = document.getElementById('issue-query').value.trim();
        if (query) {
            searchIssues(query);
        } else {
            alert('Please describe your issue before searching');
        }
    });

    // Update database button
    document.getElementById('update-db-btn').addEventListener('click', function() {
        updateDatabase();
    });

    // Allow search on Ctrl+Enter in textarea
    document.getElementById('issue-query').addEventListener('keydown', function(e) {
        if (e.key === 'Enter' && e.ctrlKey) {
            document.getElementById('search-btn').click();
        }
    });
});

function loadLastUpdated() {
    // In a real app, you would fetch this from your backend
    const now = new Date();
    document.getElementById('last-updated').textContent = now.toLocaleString();
}

function searchIssues(query) {
    // Show loading state
    const btn = document.getElementById('search-btn');
    btn.innerHTML = '<i class="fas fa-spinner fa-spin"></i> Searching...';
    btn.disabled = true;
    
    fetch('/api/issues', {
        method: 'POST',
        headers: {
            'Content-Type': 'application/json',
        },
        body: JSON.stringify({ query: query })
    })
    .then(response => response.json())
    .then(data => {
        if (data.has_results) {
            // Redirect to results page with query parameter
            window.location.href = `/results?query=${encodeURIComponent(query)}`;
        } else {
            // Offer to get AI help directly since no results found
            if (confirm('No similar issues found. Would you like AI help instead?')) {
                window.location.href = `/results?query=${encodeURIComponent(query)}&ai=1`;
            }
        }
    })
    .catch(error => {
        console.error('Error:', error);
        alert('An error occurred while searching for issues');
    })
    .finally(() => {
        btn.innerHTML = '<i class="fas fa-search"></i> Search GitHub Issues';
        btn.disabled = false;
    });
}

function updateDatabase() {
    const btn = document.getElementById('update-db-btn');
    btn.innerHTML = '<i class="fas fa-spinner fa-spin"></i> Updating...';
    btn.disabled = true;
    
    fetch('/api/update-db', {
        method: 'POST'
    })
    .then(response => response.json())
    .then(data => {
        if (data.success) {
            loadLastUpdated();
            alert('Database updated successfully!');
        } else {
            alert('Error updating database: ' + data.error);
        }
    })
    .catch(error => {
        console.error('Error:', error);
        alert('An error occurred while updating the database');
    })
    .finally(() => {
        btn.innerHTML = '<i class="fas fa-sync-alt"></i> Update Database Now';
        btn.disabled = false;
    });
}
