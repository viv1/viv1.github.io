---
# the default layout is 'page'
icon: fas fa-project-diagram
order: 5
---

<style>
.project-grid {
  display: grid;
  grid-template-columns: repeat(auto-fill, minmax(300px, 1fr));
  gap: 20px;
  margin-top: 30px;
}

.project-card {
  background: linear-gradient(145deg, #ffffff, #f5f5f5);
  border-radius: 12px;
  box-shadow: 0 4px 8px rgba(0,0,0,0.08);
  padding: 20px;
  transition: all 0.3s ease;
  position: relative;
  overflow: hidden;
  border: 1px solid rgba(0,0,0,0.05);
}

.project-card:hover {
  transform: translateY(-5px);
  box-shadow: 0 10px 20px rgba(0,0,0,0.12);
}

.project-title {
  font-size: 1.4rem;
  margin-bottom: 15px;
  font-weight: 600;
  color: #333;
  transition: color 0.3s ease;
}

.project-card:hover .project-title {
  color: #0366d6;
}

.project-description {
  color: #555;
  font-size: 0.95rem;
  line-height: 1.5;
}

.project-image-container {
  width: 60px;
  height: 60px;
  margin-bottom: 15px;
  overflow: hidden;
  border-radius: 8px;
  transition: all 0.3s ease;
}

.project-card:hover .project-image-container {
  width: 100%;
  height: 120px;
}

.project-image {
  width: 100%;
  height: 100%;
  object-fit: cover;
  transition: transform 0.3s ease;
}

.project-link {
  display: inline-block;
  margin-top: 15px;
  padding: 8px 16px;
  background-color: #0366d6;
  color: white;
  border-radius: 4px;
  text-decoration: none;
  font-size: 0.9rem;
  transition: background-color 0.3s ease;
}

.project-link:hover {
  background-color: #024ea7;
  text-decoration: none;
}

@media (max-width: 768px) {
  .project-grid {
    grid-template-columns: 1fr;
  }
}

.fade-in {
  animation: fadeIn 0.8s ease-in;
}

@keyframes fadeIn {
  from { opacity: 0; transform: translateY(20px); }
  to { opacity: 1; transform: translateY(0); }
}

.delay-1 { animation-delay: 0.1s; }
.delay-2 { animation-delay: 0.2s; }
.delay-3 { animation-delay: 0.3s; }
.delay-4 { animation-delay: 0.4s; }
.delay-5 { animation-delay: 0.5s; }
.delay-6 { animation-delay: 0.6s; }
</style>

# Side Projects & Open Source Contributions

<p>These are some of my hobby projects and open-source contributions I've worked on during my free time. Each one reflects different interests and skills.</p>

<div class="project-grid">

  <div class="project-card fade-in delay-1">
    <div class="project-image-container">
      <img src="/assets/img/projects/judgement-game.png" alt="Judgement Game" class="project-image" onerror="this.src='https://via.placeholder.com/120x60?text=Judgement+Game';this.onerror='';">
    </div>
    <h3 class="project-title">Judgement Game</h3>
    <p class="project-description">An online multiplayer card game based on the classic card game "Judgement". Play with friends or random opponents in real-time. Features include game rooms, live scoring, and interactive gameplay.</p>
    <a href="https://judgement.wewake.dev" class="project-link" target="_blank">Try it out →</a>
  </div>

  <div class="project-card fade-in delay-2">
    <div class="project-image-container">
      <img src="/assets/img/projects/check-my-returns.png" alt="Check My Returns" class="project-image" onerror="this.src='https://via.placeholder.com/120x60?text=Check+My+Returns';this.onerror='';">
    </div>
    <h3 class="project-title">Check My Returns</h3>
    <p class="project-description">A financial tool that helps investors analyze and track their investment returns. The platform provides insights, visualizations, and comparison tools to better understand investment performance.</p>
    <a href="https://checkmyreturns.in" class="project-link" target="_blank">Explore →</a>
  </div>

  <div class="project-card fade-in delay-3">
    <div class="project-image-container">
      <img src="/assets/img/projects/indexnow-submitter.png" alt="IndexNow Submitter" class="project-image" onerror="this.src='https://via.placeholder.com/120x60?text=IndexNow+Submitter';this.onerror='';">
    </div>
    <h3 class="project-title">IndexNow Submitter</h3>
    <p class="project-description">An open-source tool that helps website owners automatically submit their URLs to search engines that support the IndexNow protocol. This helps in faster indexing of web pages by search engines.</p>
    <a href="https://www.npmjs.com/package/indexnow-submitter" class="project-link" target="_blank">View on NPM →</a>
  </div>

  <div class="project-card fade-in delay-4">
    <div class="project-image-container">
      <img src="/assets/img/projects/recall-ai.png" alt="Recall AI Meeting Action Items" class="project-image" onerror="this.src='https://via.placeholder.com/120x60?text=Recall+AI';this.onerror='';">
    </div>
    <h3 class="project-title">Recall AI Meeting Action Items</h3>
    <p class="project-description">A productivity tool that integrates with Recall AI to automatically extract and organize action items from meeting recordings. It helps teams stay on top of their commitments and follow-ups.</p>
    <a href="https://github.com/viv1/recall-ai-meeting-action-items" class="project-link" target="_blank">View on GitHub →</a>
  </div>

  <div class="project-card fade-in delay-5">
    <div class="project-image-container">
      <img src="/assets/img/projects/webchat-gemini.png" alt="Web Chat for Gemini" class="project-image" onerror="this.src='https://via.placeholder.com/120x60?text=Web+Chat+Gemini';this.onerror='';">
    </div>
    <h3 class="project-title">Web Chat for Gemini - VSCode Extension</h3>
    <p class="project-description">A Visual Studio Code extension that integrates Google's Gemini AI directly into your IDE. Ask questions, get explanations, or generate code without leaving your editor.</p>
    <a href="https://marketplace.visualstudio.com/items?itemName=wewake.webchatforgemini" class="project-link" target="_blank">Get Extension →</a>
  </div>

  <div class="project-card fade-in delay-6">
    <div class="project-image-container">
      <img src="/assets/img/projects/cmy-resume.png" alt="CMy Resume" class="project-image" onerror="this.src='https://via.placeholder.com/120x60?text=CMy+Resume';this.onerror='';">
    </div>
    <h3 class="project-title">CMy Resume</h3>
    <p class="project-description">A platform to create, manage, and share professional resumes. Features customizable templates, real-time editing, and easy sharing options to streamline the job application process. [This one took a backseat, but I intend to revive it soon]</p>
    <a href="https://cmyresume.com" class="project-link" target="_blank">Visit Site →</a>
  </div>

</div>

<script>
  document.addEventListener('DOMContentLoaded', function() {
    const cards = document.querySelectorAll('.project-card');
    
    // Function to check if an element is in viewport
    function isInViewport(element) {
      const rect = element.getBoundingClientRect();
      return (
        rect.top >= 0 &&
        rect.left >= 0 &&
        rect.bottom <= (window.innerHeight || document.documentElement.clientHeight) &&
        rect.right <= (window.innerWidth || document.documentElement.clientWidth)
      );
    }
    
    // Function to handle scroll animation
    function handleScroll() {
      cards.forEach(card => {
        if (isInViewport(card) && !card.classList.contains('animated')) {
          card.classList.add('animated');
          card.style.opacity = '1';
          card.style.transform = 'translateY(0)';
        }
      });
    }
    
    // Initial check
    handleScroll();
    
    // Add scroll listener
    window.addEventListener('scroll', handleScroll);
  });
</script>