main3.py
from dotenv import load_dotenv
import os
from typing import List, Dict, Optional
from langchain_google_genai import ChatGoogleGenerativeAI
from langchain_huggingface import HuggingFaceEmbeddings
from langchain_astradb import AstraDBVectorStore
from langchain.agents import AgentExecutor, create_react_agent
from langchain.tools.retriever import create_retriever_tool
from langchain import hub
from langchain_core.documents import Document
from langchain.tools import tool
from github2 import GitHubIssueResolver
from colorama import Fore, Style, init
from colorama import Fore, Style, init
from langchain_community.tools import DuckDuckGoSearchRun
from langchain.tools import Tool
from langchain_community.utilities import SerpAPIWrapper
from langchain_core.tools import Tool


init(autoreset=True)
# Initialize colorama
init()

load_dotenv()

def display_welcome():
    """Enhanced welcome message with color"""
    print(Fore.CYAN + "=" * 60)
    print(Fore.YELLOW + "🤖 Github Issues Resolution Agent".center(60))
    print(Fore.CYAN + "=" * 60 + Style.RESET_ALL)
    print(Fore.GREEN + "\n🚀 What This Agent Does:" + Style.RESET_ALL)
    print("-" * 59)
    print(f"{Fore.BLUE}| 1. SEARCH   {Style.RESET_ALL}| Find existing GitHub issues similar to yours")
    print(f"{Fore.BLUE}| 2. SOLUTIONS{Style.RESET_ALL}| View community-approved fixes and workarounds")
    print(f"{Fore.BLUE}| 3. RESOLVE  {Style.RESET_ALL}| Comment on issues or continue as a new issues")
    print(f"{Fore.BLUE}| 4. AI HELP  {Style.RESET_ALL}| Get automated diagnosis and prevention advice")
    print("-" * 59)
    
    print(Fore.GREEN + "\n🔧 Current Configuration:" + Style.RESET_ALL)
    print(f"• Repository: {Fore.YELLOW}techwithtim/Flask-Web-App-Tutorial{Style.RESET_ALL}")
    print(f"• AI Model: {Fore.YELLOW}Gemini 1.5 Pro{Style.RESET_ALL}")
    print(f"• Database: {Fore.YELLOW}AstraDB{Style.RESET_ALL}")
    
    # print(Fore.GREEN + "\n📝 How To Proceed:" + Style.RESET_ALL)
    # print("1. Describe your issue in natural language")
    # print("2. We'll search for existing solutions")
    # print("3. Choose to comment or get new AI solution")
    # print("\n" + "=" * 60)
    # print(Fore.MAGENTA + "💡 Pro Tip: Include error messages for best results!" + Style.RESET_ALL)
    # print("=" * 60 + "\n")
def setup_web_search_tool():
    """Configure SerpAPI for reliable web searches"""
    search = SerpAPIWrapper()
    return Tool(
        name="web_search",
        func=search.run,
        description="Search the internet for technical solutions. Input should be a clear search query."
    )

@tool
def generate_solution(problem_description: str) -> str:
    """Generates technical solutions for coding problems"""
    llm = ChatGoogleGenerativeAI(
        model="gemini-1.5-pro",
        google_api_key=os.getenv("GEMINI_API_KEY")
    )
    prompt = f"""Provide a detailed solution for:
    {problem_description}
    
    Include:
    1. Root cause analysis
    2. Fixes with code examples
    3. Prevention tips"""
    
    return llm.invoke(prompt).content

def initialize_vector_store() -> AstraDBVectorStore:
    """Initialize vector store with error handling"""
    embeddings = HuggingFaceEmbeddings(model_name="all-MiniLM-L6-v2")
    # print("h2")
    # First try to connect to existing collection
    # print("API Endpoint:", os.getenv("ASTRA_DB_API_ENDPOINT"))

    try:
        return AstraDBVectorStore(

        
            embedding=embeddings,
            collection_name="issues",
            api_endpoint=os.getenv("ASTRA_DB_API_ENDPOINT"),
            token=os.getenv("ASTRA_DB_APPLICATION_TOKEN"),
            
        )
    except Exception as e:
        print(Fore.RED + f"⚠️ Error connecting to existing collection: {str(e)}" + Style.RESET_ALL)
        print(Fore.YELLOW + "Creating new collection..." + Style.RESET_ALL)
        return AstraDBVectorStore(
            embedding=embeddings,
            collection_name="issues",
            api_endpoint=os.getenv("ASTRA_DB_API_ENDPOINT"),
            token=os.getenv("ASTRA_DB_APPLICATION_TOKEN")
        )

def format_issue_response(issues: List[Document], resolution_context: Dict = None) -> str:
    """Enhanced formatting with color and better layout"""
    response = ""
    
    # if resolution_context:
    #     status = resolution_context.get('status', 'unknown').upper()
    #     color = Fore.GREEN if status == "RESOLVED" else Fore.YELLOW
    #     response += f"{color}RESOLUTION STATUS: {status}{Style.RESET_ALL}\n"
    #     if resolution_context.get("reference"):
    #         response += f"{Fore.CYAN}Reference: {resolution_context['reference']}{Style.RESET_ALL}\n\n"
    
    for i, issue in enumerate(issues, 1):
        meta = issue.metadata
        status_color = Fore.RED if meta.get('state') == 'open' else Fore.GREEN
        response += (
            f"{i}. {status_color}[{'OPEN' if meta.get('state') == 'open' else 'CLOSED'}]{Style.RESET_ALL} "
            f"{Fore.BLUE}{meta.get('title', 'Untitled')}{Style.RESET_ALL}\n"
            f"   {Fore.WHITE}#{meta.get('number')} | Created: {meta.get('created_at')}{Style.RESET_ALL}\n"
            f"   {Fore.CYAN}URL: {meta.get('url')}{Style.RESET_ALL}\n"
        )
        
        if issue.page_content:
            desc = ' '.join(line for line in issue.page_content.split('\n') 
                          if line.strip() and not line.startswith("Description:"))
            response += f"   {Fore.WHITE}Description: {desc[:200]}{'...' if len(desc) > 200 else ''}{Style.RESET_ALL}\n"
        
        if meta.get('solution'):
            response += f"   {Fore.GREEN}🔧 Solution: {meta['solution'][:150]}...{Style.RESET_ALL}\n"
        
        response += "\n"
    
    return response

def process_issue_description():
    """Main execution flow with error handling"""
    try:
        display_welcome()
        
        # Initialize components
        resolver = GitHubIssueResolver()
        vstore = initialize_vector_store()
        # print("h1")

        # # Update vector store
        # try:
        #     issues = resolver.fetch_issues_as_documents()
        #     enhanced_issues = []
            
        #     for issue in issues:
        #         resolution_data = resolver.get_issue_state(issue.metadata['number'])
        #         if resolution_data['is_resolved']:
        #             issue.metadata['solution'] = f"Fixed in PR: {resolution_data.get('pr_url', 'N/A')}"
        #         enhanced_issues.append(issue)
            
        #     vstore.add_documents(enhanced_issues)
        #     print(Fore.GREEN + f"✓ Loaded {len(enhanced_issues)} issues" + Style.RESET_ALL)
        # except Exception as e:
        #     print(Fore.RED + f"⚠️ Error loading issues: {str(e)}" + Style.RESET_ALL)
        if input("Update issues database? (y/N): ").lower() == "y":
            issues = resolver.fetch_issues_as_documents()
            try:
                vstore.delete_collection()
                # print("hii")
            except:
                pass
            vstore = initialize_vector_store()
            
            # Enhance issues with solutions from PRs/comments
            enhanced_issues = []
            for issue in issues:
                # print(issue)
                
                issue_number = issue.metadata['number']
                resolution_data = resolver.get_issue_state(issue_number)
                # print("-------")
                # print(resolution_data)
                # print("-------")
            
                if resolution_data['is_resolved']:
                    # Try to extract solution from closing PR or comments
                    solution = f"Fixed in PR: {resolution_data.get('pr_url', 'N/A')}"
                    # print(resolution_data)
                    # print(solution)
                    
                    issue.metadata['solution'] = solution
                enhanced_issues.append(issue)
            
            vstore.add_documents(enhanced_issues)
            # print(f"Loaded {len(enhanced_issues)} issues")
            print(f"Loaded issues")

        # Configure agent
        threshold=0.6
        retriever = vstore.as_retriever(search_type="similarity_score_threshold",search_kwargs={
            "k": 3,  # Return top 5 matches
            "score_threshold": threshold  # Minimum similarity score (0.0 to 1.0)
        })
        retriever_tool = create_retriever_tool(
            retriever,
            name="github_issues",
            description="Search existing GitHub issues and solutions"
        )

        llm = ChatGoogleGenerativeAI(
            model="gemini-1.5-pro",
            google_api_key=os.getenv("GEMINI_API_KEY")
        )
        web_search_tool = setup_web_search_tool()
        tools = [retriever_tool, generate_solution,web_search_tool]
        agent = create_react_agent(llm, tools, hub.pull("hwchase17/react"))
        executor = AgentExecutor(agent=agent, tools=tools, verbose=True)

        # Main interaction loop
        while True:
            try:
                user_input = input("\nDescribe your GitHub issue (or 'q' to quit): ").strip()
                if user_input.lower() == 'q':
                    break
                
                # Search existing issues
                similar_docs = retriever.invoke(user_input)
                
                if similar_docs:
                    resolution_status = resolver.get_issue_state(similar_docs[0].metadata['number'])
                    print(format_issue_response(
                        similar_docs,
                        {
                            "status": "resolved" if resolution_status["is_resolved"] else "open",
                            "reference": similar_docs[0].metadata['url']
                        }
                    ))
                    
                    action = input("\nChoose action: [1] Comment [2] New solution: ")
                    if action == "1":
                        comment = input("Enter your comment: ")
                        msg=resolver.contribute_to_issue(similar_docs[0].metadata['number'], comment)
                        print(msg)
                        continue
                    elif action == "2":
                        # Perform web search
                        print(Fore.CYAN + "\n🌐 Searching the web for solutions..." + Style.RESET_ALL)
                        result = executor.invoke({
                        "input": f"Search for technical solutions to: {user_input}. Focus on  StackOverflow and programming blogs."
                        })
                    
                        print(Fore.GREEN + "\nSearch Results:" + Style.RESET_ALL)
                        print("-" * 80)
                        print(result["output"])
                        print("-" * 80)
                    
                     
                # Generate new solution
                # print(Fore.YELLOW + "\nAnalyzing your issue..." + Style.RESET_ALL)
                # result = executor.invoke({
                #     "input": user_input,
                #     "existing_issues": format_issue_response(similar_docs) if similar_docs else "None"
                # })
                # print(Fore.GREEN + "\nSOLUTION:" + Style.RESET_ALL)
                # print(result["output"])
                
            except KeyboardInterrupt:
                print(Fore.RED + "\nOperation cancelled by user" + Style.RESET_ALL)
                break
            except Exception as e:
                print(Fore.RED + f"\n⚠️ Error: {str(e)}" + Style.RESET_ALL)

    except Exception as e:
        print(Fore.RED + f"Fatal error: {str(e)}" + Style.RESET_ALL)
    finally:
        print(Fore.CYAN + "\nThank you for using the GitHub Issue Resolution Assistant!" + Style.RESET_ALL)

-------------------------------------------------------

github2.py


# github_agent.py
import os
import requests
from typing import List, Dict, Optional
from dotenv import load_dotenv
from langchain_core.documents import Document

load_dotenv()

class GitHubIssueResolver:
    def __init__(self):
        self.headers = {
            "Authorization": f"Bearer {os.getenv('GITHUB_TOKEN')}",
            "Accept": "application/vnd.github.v3+json" #"Hey, I want my response in the format you use for version 3 of your API, and I want it as JSON."
        }
        self.owner = "techwithtim"  # Default, can be overridden
        self.repo = "Flask-Web-App-Tutorial"

    def _make_github_request(self, endpoint: str) -> Dict:
        url = f"https://api.github.com/repos/{self.owner}/{self.repo}/{endpoint}"
        response = requests.get(url, headers=self.headers)
        # print("--------")
        # print(response.json())
        return response.json() if response.status_code == 200 else {}

    def find_similar_issues(self, issue_title: str) -> List[Dict]:
        """Find issues with similar titles using GitHub search API"""
        print("hello")
        query = f"repo:{self.owner}/{self.repo} {issue_title} in:title state:all"
        print(query)
        search_url = f"https://api.github.com/search/issues?q={query}"
        response = requests.get(search_url, headers=self.headers)
        print(response.json())
        return response.json().get("items", [])

    def get_issue_state(self, issue_number: int) -> Dict:
        """Get detailed issue state including timeline events"""
        issue_data = self._make_github_request(f"issues/{issue_number}")
        # print(issue_data)
        # print("----")
        timeline_data = self._make_github_request(f"issues/{issue_number}/timeline")
        # print(timeline_data)
        # print("-----------_____")
        return {
            "basic": issue_data,
            "timeline": timeline_data,
            "is_resolved": self._check_if_resolved(issue_data, timeline_data)
        }

    def _check_if_resolved(self, issue_data: Dict, timeline_data: List) -> bool:
        """Determine if issue was properly resolved"""
        if issue_data.get("state") != "closed":
            return False
        # if timeline_data.get("event") == "closed":
        #     return True
        
        # Check for PRs that mention closing this issue
        for event in timeline_data:
            if event.get("event") == "cross-referenced":
                source = event.get("source", {})
                # print("-----------_____")
                # print(source)
                # print("-----------_____")
                if "pull_request" in source.get("html_url", ""):
                    if any(keyword in source.get("body", "").lower() 
                          for keyword in ["closes", "fixes", "resolves"]):
                        return True
        return False

    def contribute_to_issue(self, issue_number: int, comment: str) -> str:
        """Add context to existing issue"""
        # print(issue_number)
        # print(f"https://api.github.com/repos/{self.owner}/{self.repo}/issues/{issue_number}/comments")
        response = requests.post(
            f"https://api.github.com/repos/{self.owner}/{self.repo}/issues/{issue_number}/comments",
            headers=self.headers,
            json={"body": comment}
        )
        print("Status Code:", response.status_code)
        # print("Response Text:", response.text)
        return "Comment added successfully" if response.status_code == 201 else "Failed to add comment"

    def handle_new_issue(self, title: str, body: str) -> Dict:
        """Full workflow for issue handling"""
        similar_issues = self.find_similar_issues(title)
        
        for issue in similar_issues:
            issue_state = self.get_issue_state(issue["number"])
            
            # Case 1: Issue already resolved
            if issue_state["is_resolved"]:
                return {
                    "action": "duplicate",
                    "status": "resolved",
                    "message": f"This appears to be a duplicate of #{issue['number']} which was already resolved",
                    "reference": issue["html_url"]
                }
            
            # Case 2: Open issue exists
            return {
                "action": "contribute",
                "status": "open",
                "message": self.contribute_to_issue(
                    issue_number=issue["number"],
                    comment=f"Additional context from similar report:\n\n**Title**: {title}\n**Description**: {body}"
                ),
                "reference": issue["html_url"]
            }
        
        # Case 3: New issue
        return {
            "action": "new",
            "status": "unresolved",
            "message": "No similar issues found - this appears to be a new report"
        }

    def fetch_issues_as_documents(self) -> List[Document]:
        """Fetch all issues as LangChain Documents for vector store"""
        issues = self._make_github_request("issues?state=all")
        docs = []
        # print("__________________")
       # print(issues[0])
        for issue in issues:
            
            
            metadata = {
                "number": issue["number"],
                "title": issue["title"],

                "url": issue["html_url"],
                "state": issue["state"],
                "created_at": issue["created_at"]
            }
            content = f"Issue #{issue['number']}: {issue['title']}\nState: {issue['state']}\n"
            content += f"Description:\n{issue['body']}\n" if issue.get("body") else ""
            
            docs.append(Document(page_content=content, metadata=metadata))
            # print("----------------n")
        # print(docs)
        return docs

----------------------------------------------------------

results.html



 <!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Search Results - GitHub Issue Resolver</title>
    <link rel="stylesheet" href="{{ url_for('static', filename='css/results.css') }}">
    <link href="https://fonts.googleapis.com/css2?family=Roboto:wght@300;400;500;700&display=swap" rel="stylesheet">
    <link rel="stylesheet" href="https://cdnjs.cloudflare.com/ajax/libs/font-awesome/6.0.0/css/all.min.css">
</head>
<body>
    <div class="container">
        <header>
            <h1><i class="fab fa-github"></i> Issue Results</h1>
            <a href="/" class="back-btn"><i class="fas fa-arrow-left"></i> New Search</a>
        </header>

        <main>
            <div class="query-display">
                <h2>Search results for: <span id="search-query"></span></h2>
            </div>

         
            <section class="results-section" id="github-results">
                <h3><i class="fas fa-code-branch"></i> GitHub Issues</h3>
                <div class="results-container" id="issues-container">
                    <div class="loading-spinner">
                        <i class="fas fa-spinner fa-spin"></i> Loading GitHub issues...
                    </div>
                </div>
            </section>

          
            <section class="ai-help-section" id="ai-help-section">
                <div class="ai-help-card">
                    <h3><i class="fas fa-robot"></i> Need More Help?</h3>
                    <p>If the GitHub results didn't solve your problem, try our AI assistant:</p>
                    <button id="ai-help-btn" class="accent-btn">
                        <i class="fas fa-magic"></i> Get AI Assistance
                    </button>
                </div>
            </section>

            
            <section class="ai-solution-section" id="ai-solution-section" style="display: none;">
                <h3><i class="fas fa-robot"></i> AI-Generated Solution</h3>
                <div class="ai-solution" id="ai-solution">
                    <div class="loading-spinner">
                        <i class="fas fa-spinner fa-spin"></i> Generating AI solution...
                    </div>
                </div>
            </section>
        </main>
    </div>

    <script src="{{ url_for('static', filename='js/results.js') }}"></script>
</body> 
</html> 


----------------------------------------------

results.css


/* Inherit base styles from main CSS */
@import url('style.css');

/* Results Page Specific Styles */
.container {
    max-width: 900px;
}

header {
    display: flex;
    justify-content: space-between;
    align-items: center;
    margin-bottom: 30px;
}

.back-btn {
    background-color: var(--light-color);
    color: var(--dark-color);
    padding: 10px 15px;
    border-radius: 4px;
    text-decoration: none;
    display: inline-flex;
    align-items: center;
    gap: 5px;
    transition: background-color 0.3s;
}

.back-btn:hover {
    background-color: #d5dbdb;
}

.query-display {
    background: white;
    padding: 15px 20px;
    border-radius: 8px;
    box-shadow: 0 2px 4px rgba(0, 0, 0, 0.05);
    margin-bottom: 20px;
}

.query-display h2 {
    margin: 0;
    font-size: 1.3rem;
}

.query-display span {
    font-weight: normal;
    color: var(--secondary-color);
}

.results-section, .ai-help-section, .ai-solution-section {
    background: white;
    padding: 20px;
    border-radius: 8px;
    box-shadow: 0 2px 4px rgba(0, 0, 0, 0.05);
    margin-bottom: 20px;
}

.results-section h3, .ai-help-section h3, .ai-solution-section h3 {
    margin-top: 0;
    color: var(--primary-color);
    display: flex;
    align-items: center;
    gap: 10px;
    padding-bottom: 10px;
    border-bottom: 1px solid #eee;
}

.ai-help-card {
    text-align: center;
    padding: 20px;
}

.ai-help-card p {
    margin-bottom: 20px;
}

.accent-btn {
    background-color: var(--accent-color);
    color: white;
    padding: 12px 25px;
    font-size: 1.1rem;
}

.accent-btn:hover {
    background-color: #c0392b;
}

.loading-spinner {
    color: #666;
    text-align: center;
    padding: 20px;
}

/* Issue Cards */
.issue-card {
    background: #f8f9fa;
    padding: 15px;
    border-radius: 4px;
    border-left: 4px solid var(--secondary-color);
    margin-bottom: 15px;
}

.issue-header {
    display: flex;
    justify-content: space-between;
    align-items: center;
    margin-bottom: 10px;
}

.issue-title {
    font-weight: 500;
    color: var(--primary-color);
    margin: 0;
    font-size: 1.1rem;
}

.issue-state {
    padding: 3px 10px;
    border-radius: 12px;
    font-size: 0.8rem;
    font-weight: 500;
}

.issue-state.open {
    background-color: #f39c12;
    color: white;
}

.issue-state.closed {
    background-color: #2ecc71;
    color: white;
}

.issue-meta {
    display: flex;
    gap: 15px;
    font-size: 0.9rem;
    color: #666;
    margin-bottom: 10px;
    flex-wrap: wrap;
}

.issue-description {
    margin-bottom: 10px;
    line-height: 1.5;
    white-space: pre-line;
}

.issue-solution {
    background: #e8f4fd;
    padding: 12px;
    border-radius: 4px;
    margin-top: 15px;
    border-left: 3px solid var(--info-color);
}

.issue-solution h4 {
    margin-top: 0;
    color: var(--info-color);
    font-size: 1rem;
}

.issue-actions {
    display: flex;
    gap: 10px;
    margin-top: 15px;
}

.view-issue-btn, .comment-btn {
    padding: 8px 15px;
    font-size: 0.9rem;
    text-decoration: none;
}

.view-issue-btn {
    background-color: var(--secondary-color);
    color: white;
}

.comment-btn {
    background-color: var(--light-color);
    color: var(--dark-color);
}

/* Responsive Design */
@media (max-width: 768px) {
    header {
        flex-direction: column;
        align-items: flex-start;
        gap: 10px;
    }
    
    .issue-header {
        flex-direction: column;
        align-items: flex-start;
        gap: 5px;
    }
    
    .issue-meta {
        flex-direction: column;
        gap: 5px;
    }
    
    .issue-actions {
        flex-direction: column;
    }
}
------------------------------

results.js


// if (document.readyState === "loading") {
// document.addEventListener('DOMContentLoaded', function() {
//   // Get query parameters
//   console.log("dom fully lpaded")
//   const urlParams = new URLSearchParams(window.location.search);
//   const query = urlParams.get('query');
//   const aiRequested = urlParams.has('ai');
  
//   // Display the search query
//   document.getElementById('search-query').textContent = query;
  
//   // Load GitHub issues
//   loadGitHubIssues(query);
  
//   // Set up AI help buttonconst aiHelpBtn = document.getElementById('ai-help-btn');
//   const aiHelpBtn = document.getElementById('ai-help-btn');
//   console.log("AI Help button element:", aiHelpBtn);
//   if (!aiHelpBtn) {
//     console.error("Error: AI Help button not found in DOM!");
//     return;
//   }
//   document.getElementById('ai-help-btn').addEventListener('click', function() {
//     console.log("AI Help button clicked");
//       getAISolution(query);
//   });
  
//   // If AI was requested directly, show that section immediately
//   if (aiRequested) {
//       document.getElementById('ai-help-section').style.display = 'none';
//       document.getElementById('ai-solution-section').style.display = 'block';
//       getAISolution(query);
//   }
// });

// function loadGitHubIssues(query) {
//   const issuesContainer = document.getElementById('issues-container');
  
//   fetch('/api/issues', {
//       method: 'POST',
//       headers: {
//           'Content-Type': 'application/json',
//       },
//       body: JSON.stringify({ query: query })
//   })
//   .then(response => response.json())
//   .then(data => {
//       if (data.has_results) {
//           displayGitHubIssues(data.issues);
//       } else {
//           issuesContainer.innerHTML = `
//               <div class="no-results">
//                   <p>No similar GitHub issues found for this query.</p>
//                   <p>Try our AI assistant for help with this issue.</p>
//               </div>
//           `;
//       }
//   })
//   .catch(error => {
//       console.error('Error:', error);
//       issuesContainer.innerHTML = `
//           <div class="error">
//               <p>Error loading GitHub issues. Please try again.</p>
//           </div>
//       `;
//   });
// }

// function displayGitHubIssues(issues) {
//   const issuesContainer = document.getElementById('issues-container');
  
//   if (!issues || issues.length === 0) {
//       issuesContainer.innerHTML = `
//           <div class="no-results">
//               <p>No matching GitHub issues found.</p>
//           </div>
//       `;
//       return;
//   }
  
//   issuesContainer.innerHTML = '';
  
//   issues.forEach(issue => {
//       const issueCard = document.createElement('div');
//       issueCard.className = 'issue-card';
      
//       issueCard.innerHTML = `
//           <div class="issue-header">
//               <h4 class="issue-title">${issue.title}</h4>
//               <span class="issue-state ${issue.state === 'open' ? 'open' : 'closed'}">
//                   ${issue.state}
//               </span>
//           </div>
//           <div class="issue-meta">
//               <span>#${issue.number}</span>
//               <span>Created: ${new Date(issue.created_at).toLocaleDateString()}</span>
//               <a href="${issue.url}" target="_blank">View on GitHub</a>
//           </div>
//           <div class="issue-description">
//               ${issue.description}
//           </div>
//           ${issue.solution ? `
//               <div class="issue-solution">
//                   <h4>Solution</h4>
//                   <p>${issue.solution}</p>
//               </div>
//           ` : ''}
//           <div class="issue-actions">
//               <a href="${issue.url}" target="_blank" class="view-issue-btn">
//                   <i class="fas fa-external-link-alt"></i> View Issue
//               </a>
//               <button class="comment-btn" data-issue-number="${issue.number}">
//                   <i class="fas fa-comment"></i> Add Comment
//               </button>
//           </div>
//       `;
      
//       issuesContainer.appendChild(issueCard);
//   });
  
//   // Add event listeners to comment buttons
//   document.querySelectorAll('.comment-btn').forEach(button => {
//       button.addEventListener('click', function() {
//           const issueNumber = this.getAttribute('data-issue-number');
//           addComment(issueNumber);
//       });
//   });
// }

// // function getAISolution(query) {
// //   console.log("hii")
// //   const aiHelpSection = document.getElementById('ai-help-section');
// //   const aiSolutionSection = document.getElementById('ai-solution-section');
// //   const aiSolution = document.getElementById('ai-solution');
// //   aiSolution.innerHTML = '<p class="loading"><i class="fas fa-spinner fa-spin"></i> Generating solution...</p>';
// //   // Hide help prompt, show solution section
// //   aiHelpSection.style.display = 'none';
// //   aiSolutionSection.style.display = 'block';
// //   console.log("Query being sent to AI:", query); // Add at start of getAISolution
// //   // Scroll to AI solution
// //   aiSolutionSection.scrollIntoView({ behavior: 'smooth' });
  
// //   fetch('/api/solution', {
// //       method: 'POST',
// //       headers: {
// //           'Content-Type': 'application/json',
// //       },
// //       body: JSON.stringify({ problem: query })
// //   })
// //   .then(response => response.json())
// //   .then(data => {
// //       if (data.solution) {
// //           aiSolution.innerHTML = `
// //               <div class="markdown-content">
// //                   ${data.solution}
// //               </div>
// //               <div class="feedback-buttons">
// //                   <button id="solution-helpful" class="feedback-btn helpful">
// //                       <i class="fas fa-thumbs-up"></i> Helpful
// //                   </button>
// //                   <button id="solution-not-helpful" class="feedback-btn not-helpful">
// //                       <i class="fas fa-thumbs-down"></i> Not Helpful
// //                   </button>
// //               </div>
// //           `;
          
// //           // Add feedback event listeners
// //           document.getElementById('solution-helpful').addEventListener('click', () => {
// //               alert('Thanks for your feedback!');
// //           });
          
// //           document.getElementById('solution-not-helpful').addEventListener('click', () => {
// //               const feedback = prompt('What could be improved about this solution?');
// //               if (feedback) {
// //                   alert('Thanks for your feedback! We\'ll use it to improve our answers.');
// //               }
// //           });
// //       } else {
// //           aiSolution.innerHTML = `
// //               <div class="error">
// //                   <p>Error generating AI solution. Please try again later.</p>
// //                   ${data.error ? `<p>${data.error}</p>` : ''}
// //               </div>
// //           `;
// //       }
// //   })
// //   .catch(error => {
// //       console.error('Error:', error);
// //       aiSolution.innerHTML = `
// //           <div class="error">
// //               <p>Error generating AI solution. Please try again later.</p>
// //           </div>
// //       `;
// //   });
// // }

// function getAISolution(query) {
//     console.log("Fetching AI solution...");
//     const aiHelpSection = document.getElementById('ai-help-section');
//     const aiSolutionSection = document.getElementById('ai-solution-section');
//     const aiSolution = document.getElementById('ai-solution');

//     // Show loading spinner and hide the help section
//     aiSolution.innerHTML = '<p class="loading"><i class="fas fa-spinner fa-spin"></i> Generating solution...</p>';
//     aiHelpSection.style.display = 'none';
//     aiSolutionSection.style.display = 'block';

//     // Scroll to the AI solution section
//     aiSolutionSection.scrollIntoView({ behavior: 'smooth' });

//     // Fetch AI solution and web search links
//     fetch('/api/solution', {
//         method: 'POST',
//         headers: {
//             'Content-Type': 'application/json',
//         },
//         body: JSON.stringify({ problem: query })
//     })
//     .then(response => response.json())
//     .then(data => {
//         if (data.links || data.solution) {
//             // Display links to websites
//             const linksHTML = data.links
//                 ? `<h4><i class="fas fa-search"></i> Relevant Links</h4>
//                    <ul>
//                        ${data.links.split('\n').map(link => {
//                            if (link.includes('URL: ')) {
//                                const [title, url] = link.split('URL: ');
//                                return `<li><a href="${url.trim()}" target="_blank">${title.trim()}</a></li>`;
//                            }
//                            return ''; // Skip invalid lines
//                        }).join('')}
//                    </ul>`
//                 : '<p>No web search results found.</p>';

//             // Display AI-generated summary
//             const summaryHTML = data.solution
//                 ? `<h4><i class="fas fa-lightbulb"></i> Summary</h4>
//                    <p>${data.solution}</p>`
//                 : '<p>No AI solution generated.</p>';

//             // Combine links and summary into the AI solution section
//             aiSolution.innerHTML = `
//                 <div class="ai-links">
//                     ${linksHTML}
//                 </div>
//                 <div class="ai-summary" style="margin-top: 20px;">
//                     ${summaryHTML}
//                 </div>
//                 <div class="feedback-buttons" style="margin-top: 20px;">
//                     <button id="solution-helpful" class="feedback-btn helpful">
//                         <i class="fas fa-thumbs-up"></i> Helpful
//                     </button>
//                     <button id="solution-not-helpful" class="feedback-btn not-helpful">
//                         <i class="fas fa-thumbs-down"></i> Not Helpful
//                     </button>
//                 </div>
//             `;

//             // Add feedback event listeners
//             document.getElementById('solution-helpful').addEventListener('click', () => {
//                 alert('Thanks for your feedback!');
//             });

//             document.getElementById('solution-not-helpful').addEventListener('click', () => {
//                 const feedback = prompt('What could be improved about this solution?');
//                 if (feedback) {
//                     alert('Thanks for your feedback! We\'ll use it to improve our answers.');
//                 }
//             });
//         } else {
//             aiSolution.innerHTML = `
//                 <div class="error">
//                     <p>No results found. Please try again later.</p>
//                 </div>
//             `;
//         }
//     })
//     .catch(error => {
//         console.error('Error fetching AI solution:', error);
//         aiSolution.innerHTML = `
//             <div class="error">
//                 <p>Error generating AI solution. Please try again later.</p>
//             </div>
//         `;
//     });
// }
// function addComment(issueNumber) {
//   const comment = prompt('Enter your comment to add to this issue:');
//   if (comment && comment.trim()) {
//       fetch('/api/comment', {
//           method: 'POST',
//           headers: {
//               'Content-Type': 'application/json',
//           },
//           body: JSON.stringify({
//               issue_number: issueNumber,
//               comment: comment
//           })
//       })
//       .then(response => response.json())
//       .then(data => {
//           alert(data.message || 'Comment added successfully!');
//       })
//       .catch(error => {
//           console.error('Error:', error);
//           alert('Error adding comment');
//       });
//   }
// }
// }

// else {
//     alert("DOM already loaded");
// }


if (document.readyState === "loading") {
    document.addEventListener('DOMContentLoaded', function () {
        console.log("DOM fully loaded");

        // Get query parameters
        const urlParams = new URLSearchParams(window.location.search);
        const query = urlParams.get('query');
        const aiRequested = urlParams.has('ai');

        // Display the search query
        document.getElementById('search-query').textContent = query;

        // Load GitHub issues
        loadGitHubIssues(query);

        // Set up AI help button
        const aiHelpBtn = document.getElementById('ai-help-btn');
        if (!aiHelpBtn) {
            console.error("Error: AI Help button not found in DOM!");
            return;
        }

        aiHelpBtn.addEventListener('click', function () {
            console.log("AI Help button clicked");
            getAISolution(query);
        });

        // If AI was requested directly, show that section immediately
        if (aiRequested) {
            document.getElementById('ai-help-section').style.display = 'none';
            document.getElementById('ai-solution-section').style.display = 'block';
            getAISolution(query);
        }
    });
} else {
    alert("DOM already loaded");
}

// Function to load GitHub issues
function loadGitHubIssues(query) {
    const issuesContainer = document.getElementById('issues-container');

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
                displayGitHubIssues(data.issues);
            } else {
                issuesContainer.innerHTML = `
                    <div class="no-results">
                        <p>No similar GitHub issues found for this query.</p>
                        <p>Try our AI assistant for help with this issue.</p>
                    </div>
                `;
            }
        })
        .catch(error => {
            console.error('Error:', error);
            issuesContainer.innerHTML = `
                <div class="error">
                    <p>Error loading GitHub issues. Please try again.</p>
                </div>
            `;
        });
}

// Function to display GitHub issues
function displayGitHubIssues(issues) {
    const issuesContainer = document.getElementById('issues-container');

    if (!issues || issues.length === 0) {
        issuesContainer.innerHTML = `
            <div class="no-results">
                <p>No matching GitHub issues found.</p>
            </div>
        `;
        return;
    }

    issuesContainer.innerHTML = '';

    issues.forEach(issue => {
        const issueCard = document.createElement('div');
        issueCard.className = 'issue-card';

        issueCard.innerHTML = `
            <div class="issue-header">
                <h4 class="issue-title">${issue.title}</h4>
                <span class="issue-state ${issue.state === 'open' ? 'open' : 'closed'}">
                    ${issue.state}
                </span>
            </div>
            <div class="issue-meta">
                <span>#${issue.number}</span>
                <span>Created: ${new Date(issue.created_at).toLocaleDateString()}</span>
                <a href="${issue.url}" target="_blank">View on GitHub</a>
            </div>
            <div class="issue-description">
                ${issue.description}
            </div>
            ${issue.solution ? `
                <div class="issue-solution">
                    <h4>Solution</h4>
                    <p>${issue.solution}</p>
                </div>
            ` : ''}
            <div class="issue-actions">
                <a href="${issue.url}" target="_blank" class="view-issue-btn">
                    <i class="fas fa-external-link-alt"></i> View Issue
                </a>
                <button class="comment-btn" data-issue-number="${issue.number}">
                    <i class="fas fa-comment"></i> Add Comment
                </button>
            </div>
        `;

        issuesContainer.appendChild(issueCard);
    });

    // Add event listeners to comment buttons
    document.querySelectorAll('.comment-btn').forEach(button => {
        button.addEventListener('click', function () {
            const issueNumber = this.getAttribute('data-issue-number');
            addComment(issueNumber);
        });
    });
}

// Function to fetch and display AI solution
function getAISolution(query) {
    console.log("Fetching AI solution...");
    const aiHelpSection = document.getElementById('ai-help-section');
    const aiSolutionSection = document.getElementById('ai-solution-section');
    const aiSolution = document.getElementById('ai-solution');

    // Show loading spinner and hide the help section
    aiSolution.innerHTML = '<p class="loading"><i class="fas fa-spinner fa-spin"></i> Generating solution...</p>';
    aiHelpSection.style.display = 'none';
    aiSolutionSection.style.display = 'block';

    // Scroll to the AI solution section
    aiSolutionSection.scrollIntoView({ behavior: 'smooth' });

    // Fetch AI solution and web search links
    fetch('/api/solution', {
        method: 'POST',
        headers: {
            'Content-Type': 'application/json',
        },
        body: JSON.stringify({ problem: query })
    })
        .then(response => response.json())
        .then(data => {
            console.log("Links data received:", data.links); // Debugging

            if (data.links || data.solution) {
                // Filter and display links to websites with numbering
                const linksHTML = data.links
                    ? `<h4><i class="fas fa-search"></i> Relevant Links</h4>
                       <ol>
                           ${data.links.split('\n')
                               .filter(link => link.includes('URL: ')) // Filter valid lines
                               .map((link, index) => {
                                   const [message, url] = link.split('URL: ');
                                   return `<li>${index + 1}. ${message.trim()} <a href="${url.trim()}" target="_blank">${url.trim()}</a></li>`;
                               }).join('')}
                       </ol>`
                    : '<p>No web search results found.</p>';

                // Display AI-generated summary
                const summaryHTML = data.solution
                    ? `<h4><i class="fas fa-lightbulb"></i> Summary</h4>
                       <p>${data.solution}</p>`
                    : '<p>No AI solution generated.</p>';

                // Combine links and summary into the AI solution section
                aiSolution.innerHTML = `
                    <div class="ai-links">
                        ${linksHTML}
                    </div>
                    <div class="ai-summary" style="margin-top: 20px;">
                        ${summaryHTML}
                    </div>
                `;
            } else {
                aiSolution.innerHTML = `
                    <div class="error">
                        <p>No results found. Please try again later.</p>
                    </div>
                `;
            }
        })
        .catch(error => {
            console.error('Error fetching AI solution:', error);
            aiSolution.innerHTML = `
                <div class="error">
                    <p>Error generating AI solution. Please try again later.</p>
                </div>
            `;
        });
}
// Function to add a comment to a GitHub issue
function addComment(issueNumber) {
    const comment = prompt('Enter your comment to add to this issue:');
    if (comment && comment.trim()) {
        fetch('/api/comment', {
            method: 'POST',
            headers: {
                'Content-Type': 'application/json',
            },
            body: JSON.stringify({
                issue_number: issueNumber,
                comment: comment
            })
        })
            .then(response => response.json())
            .then(data => {
                alert(data.message || 'Comment added successfully!');
            })
            .catch(error => {
                console.error('Error:', error);
                alert('Error adding comment');
            });
    }
}


