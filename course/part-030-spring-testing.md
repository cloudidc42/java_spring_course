# Part 030: Spring Testing

Testing is a first-class concern in Spring Boot. The framework provides powerful tools for testing individual layers or the full stack with minimal configuration.

---

## 1. Testing Strategy

```
Testing Pyramid:
                /\
               /  \
              / E2E \          (Few - slow, expensive)
             /--------\
            / Integration \    (Some - medium speed)
           /--------------\
          /   Unit Tests    \  (Many - fast, cheap)
         /------------------\

Spring Boot Test Slices:
- @WebMvcTest    -> Controller layer only
- @DataJpaTest   -> Repository layer only
- @SpringBootTest -> Full application context
```

---

## 2. Maven Dependencies

```xml
<dependencies>
    <!-- Spring Boot Test (includes JUnit 5, Mockito, MockMvc) -->
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-test</artifactId>
        <scope>test</scope>
    </dependency>

    <!-- Spring Security Test -->
    <dependency>
        <groupId>org.springframework.security</groupId>
        <artifactId>spring-security-test</artifactId>
        <scope>test</scope>
    </dependency>

    <!-- TestContainers -->
    <dependency>
        <groupId>org.testcontainers</groupId>
        <artifactId>junit-jupiter</artifactId>
        <scope>test</scope>
    </dependency>
    <dependency>
        <groupId>org.testcontainers</groupId>
        <artifactId>postgresql</artifactId>
        <scope>test</scope>
    </dependency>

    <!-- RestAssured for integration tests -->
    <dependency>
        <groupId>io.rest-assured</groupId>
        <artifactId>rest-assured</artifactId>
        <scope>test</scope>
    </dependency>

    <!-- AssertJ (included in spring-boot-starter-test but explicit) -->
    <dependency>
        <groupId>org.assertj</groupId>
        <artifactId>assertj-core</artifactId>
        <scope>test</scope>
    </dependency>
</dependencies>
```

---

## 3. Unit Testing with Mockito

### Service Unit Tests

```java
package com.example.blog.service;

import com.example.blog.entity.Post;
import com.example.blog.entity.PostStatus;
import com.example.blog.entity.User;
import com.example.blog.exception.PostNotFoundException;
import com.example.blog.repository.PostRepository;
import com.example.blog.repository.UserRepository;
import org.junit.jupiter.api.*;
import org.junit.jupiter.api.extension.ExtendWith;
import org.mockito.*;
import org.mockito.junit.jupiter.MockitoExtension;

import java.time.LocalDateTime;
import java.util.List;
import java.util.Optional;

import static org.assertj.core.api.Assertions.*;
import static org.mockito.ArgumentMatchers.*;
import static org.mockito.BDDMockito.*;

@ExtendWith(MockitoExtension.class)  // Initialize mocks without Spring context
@DisplayName("PostService Unit Tests")
class PostServiceTest {

    @Mock
    private PostRepository postRepository;

    @Mock
    private UserRepository userRepository;

    @Mock
    private SlugUtils slugUtils;

    @InjectMocks   // Inject mocks into PostService via constructor/setter/field
    private PostService postService;

    // Test fixtures
    private User testUser;
    private Post testPost;

    @BeforeEach
    void setUp() {
        testUser = User.builder()
            .id(1L)
            .username("testuser")
            .email("test@example.com")
            .build();

        testPost = Post.builder()
            .id(1L)
            .title("Test Post")
            .slug("test-post")
            .content("Test content")
            .status(PostStatus.DRAFT)
            .author(testUser)
            .createdAt(LocalDateTime.now())
            .updatedAt(LocalDateTime.now())
            .build();
    }

    @Test
    @DisplayName("findById should return post when it exists")
    void findById_shouldReturnPost_whenExists() {
        // Arrange (Given)
        given(postRepository.findById(1L)).willReturn(Optional.of(testPost));

        // Act (When)
        Optional<Post> result = postService.findById(1L);

        // Assert (Then)
        assertThat(result).isPresent();
        assertThat(result.get().getId()).isEqualTo(1L);
        assertThat(result.get().getTitle()).isEqualTo("Test Post");

        // Verify the mock was called
        then(postRepository).should(times(1)).findById(1L);
    }

    @Test
    @DisplayName("findById should return empty when post not found")
    void findById_shouldReturnEmpty_whenNotFound() {
        given(postRepository.findById(99L)).willReturn(Optional.empty());

        Optional<Post> result = postService.findById(99L);

        assertThat(result).isEmpty();
    }

    @Test
    @DisplayName("createPost should save and return post")
    void createPost_shouldSaveAndReturnPost() {
        // Arrange
        CreatePostRequest request = CreatePostRequest.builder()
            .title("New Post Title")
            .content("New post content")
            .excerpt("Short excerpt")
            .build();

        given(userRepository.findByUsername("testuser")).willReturn(Optional.of(testUser));
        given(slugUtils.generateUniqueSlug(eq("New Post Title"), any())).willReturn("new-post-title");
        given(postRepository.save(any(Post.class))).willAnswer(invocation -> {
            Post post = invocation.getArgument(0);
            post.setId(2L);  // Simulate database saving
            return post;
        });

        // Act
        Post result = postService.createPost(request, "testuser");

        // Assert
        assertThat(result).isNotNull();
        assertThat(result.getId()).isEqualTo(2L);
        assertThat(result.getTitle()).isEqualTo("New Post Title");
        assertThat(result.getSlug()).isEqualTo("new-post-title");
        assertThat(result.getStatus()).isEqualTo(PostStatus.DRAFT);
        assertThat(result.getAuthor().getUsername()).isEqualTo("testuser");

        // Verify repository was called
        ArgumentCaptor<Post> postCaptor = ArgumentCaptor.forClass(Post.class);
        then(postRepository).should(times(1)).save(postCaptor.capture());
        assertThat(postCaptor.getValue().getTitle()).isEqualTo("New Post Title");
    }

    @Test
    @DisplayName("publishPost should update status to PUBLISHED")
    void publishPost_shouldUpdateStatus() {
        given(postRepository.findById(1L)).willReturn(Optional.of(testPost));
        given(postRepository.save(any(Post.class))).willAnswer(inv -> inv.getArgument(0));

        Post result = postService.publishPost(1L);

        assertThat(result.getStatus()).isEqualTo(PostStatus.PUBLISHED);
        assertThat(result.getPublishedAt()).isNotNull();
    }

    @Test
    @DisplayName("publishPost should throw when post not found")
    void publishPost_shouldThrow_whenNotFound() {
        given(postRepository.findById(99L)).willReturn(Optional.empty());

        assertThatThrownBy(() -> postService.publishPost(99L))
            .isInstanceOf(PostNotFoundException.class)
            .hasMessageContaining("99");
    }

    @Test
    @DisplayName("publishPost should throw when already published")
    void publishPost_shouldThrow_whenAlreadyPublished() {
        testPost.setStatus(PostStatus.PUBLISHED);
        given(postRepository.findById(1L)).willReturn(Optional.of(testPost));

        assertThatThrownBy(() -> postService.publishPost(1L))
            .isInstanceOf(IllegalStateException.class)
            .hasMessageContaining("already published");
    }

    @Test
    @DisplayName("deletePost should call repository delete")
    void deletePost_shouldCallRepositoryDelete() {
        given(postRepository.findById(1L)).willReturn(Optional.of(testPost));
        willDoNothing().given(postRepository).delete(testPost);

        postService.deletePost(1L, "testuser");

        then(postRepository).should(times(1)).delete(testPost);
    }

    @Test
    @DisplayName("deletePost should throw when user is not owner")
    void deletePost_shouldThrow_whenNotOwner() {
        given(postRepository.findById(1L)).willReturn(Optional.of(testPost));

        assertThatThrownBy(() -> postService.deletePost(1L, "otheruser"))
            .isInstanceOf(AccessDeniedException.class);

        then(postRepository).should(never()).delete(any());
    }

    @Test
    @DisplayName("findByStatus returns list of posts")
    void findByStatus_returnsFilteredPosts() {
        List<Post> publishedPosts = List.of(
            Post.builder().id(1L).status(PostStatus.PUBLISHED).build(),
            Post.builder().id(2L).status(PostStatus.PUBLISHED).build()
        );
        given(postRepository.findByStatus(PostStatus.PUBLISHED))
            .willReturn(publishedPosts);

        List<Post> result = postService.findByStatus(PostStatus.PUBLISHED);

        assertThat(result).hasSize(2);
        assertThat(result).allMatch(p -> p.getStatus() == PostStatus.PUBLISHED);
    }
}
```

### Spy Example

```java
@ExtendWith(MockitoExtension.class)
class SlugUtilsTest {

    @Spy
    private SlugUtils slugUtils;  // Spy: real object with ability to override specific methods

    @Test
    void toSlug_shouldConvertTitleToSlug() {
        // Spy calls real method
        String slug = slugUtils.toSlug("Hello World! This is a Test");
        assertThat(slug).isEqualTo("hello-world-this-is-a-test");
    }

    @Test
    void generateUniqueSlug_whenSlugExists_shouldAppendCounter() {
        // Spy: override existsCheck to simulate existing slug
        Predicate<String> existsCheck = slug -> slug.equals("hello-world");

        String result = slugUtils.generateUniqueSlug("Hello World", existsCheck);

        assertThat(result).isEqualTo("hello-world-1");
    }
}
```

---

## 4. @WebMvcTest - Controller Layer Tests

```java
package com.example.blog.controller;

import com.example.blog.entity.Post;
import com.example.blog.entity.PostStatus;
import com.example.blog.security.JwtAuthenticationFilter;
import com.example.blog.security.JwtTokenService;
import com.example.blog.service.PostService;
import com.fasterxml.jackson.databind.ObjectMapper;
import org.junit.jupiter.api.*;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.test.autoconfigure.web.servlet.WebMvcTest;
import org.springframework.boot.test.mock.mockito.MockBean;
import org.springframework.data.domain.Page;
import org.springframework.data.domain.PageImpl;
import org.springframework.data.domain.PageRequest;
import org.springframework.http.MediaType;
import org.springframework.security.test.context.support.WithMockUser;
import org.springframework.test.web.servlet.MockMvc;

import java.time.LocalDateTime;
import java.util.List;
import java.util.Optional;

import static org.hamcrest.Matchers.*;
import static org.mockito.ArgumentMatchers.*;
import static org.mockito.BDDMockito.*;
import static org.springframework.security.test.web.servlet.request.SecurityMockMvcRequestPostProcessors.csrf;
import static org.springframework.test.web.servlet.request.MockMvcRequestBuilders.*;
import static org.springframework.test.web.servlet.result.MockMvcResultHandlers.print;
import static org.springframework.test.web.servlet.result.MockMvcResultMatchers.*;

// @WebMvcTest loads only web layer: controllers, filters, security
// Does NOT load: services, repositories, @Component beans (use @MockBean for those)
@WebMvcTest(PostController.class)
@DisplayName("PostController Tests")
class PostControllerTest {

    @Autowired
    private MockMvc mockMvc;

    @Autowired
    private ObjectMapper objectMapper;

    // Mock dependencies not loaded by @WebMvcTest
    @MockBean
    private PostService postService;

    @MockBean
    private JwtTokenService jwtTokenService;  // Needed for security config

    @MockBean
    private CustomUserDetailsService userDetailsService;

    private Post testPost;
    private PostDTOV2 testPostDTO;

    @BeforeEach
    void setUp() {
        User author = User.builder().id(1L).username("testuser").email("test@example.com").build();

        testPost = Post.builder()
            .id(1L)
            .title("Test Post")
            .slug("test-post")
            .content("Test content here")
            .excerpt("Short excerpt")
            .status(PostStatus.PUBLISHED)
            .author(author)
            .viewCount(100)
            .publishedAt(LocalDateTime.of(2024, 1, 15, 10, 0))
            .createdAt(LocalDateTime.of(2024, 1, 15, 9, 0))
            .updatedAt(LocalDateTime.of(2024, 1, 15, 9, 0))
            .build();

        testPostDTO = PostDTOV2.from(testPost);
    }

    // ================================
    // GET /api/v2/posts
    // ================================

    @Test
    @DisplayName("GET /api/v2/posts - should return paginated posts")
    void getPosts_shouldReturnPaginatedPosts() throws Exception {
        Page<Post> page = new PageImpl<>(List.of(testPost), PageRequest.of(0, 10), 1);
        given(postService.findAllPublished(any())).willReturn(page);

        mockMvc.perform(get("/api/v2/posts")
                .contentType(MediaType.APPLICATION_JSON))
            .andDo(print())
            .andExpect(status().isOk())
            .andExpect(content().contentType(MediaType.APPLICATION_JSON))
            .andExpect(jsonPath("$.data", hasSize(1)))
            .andExpect(jsonPath("$.data[0].id").value(1))
            .andExpect(jsonPath("$.data[0].title").value("Test Post"))
            .andExpect(jsonPath("$.data[0].slug").value("test-post"))
            .andExpect(jsonPath("$.pagination.totalElements").value(1))
            .andExpect(jsonPath("$.pagination.totalPages").value(1));
    }

    @Test
    @DisplayName("GET /api/v2/posts - should handle pagination parameters")
    void getPosts_shouldHandlePagination() throws Exception {
        Page<Post> emptyPage = new PageImpl<>(List.of(), PageRequest.of(2, 5), 10);
        given(postService.findAllPublished(any())).willReturn(emptyPage);

        mockMvc.perform(get("/api/v2/posts")
                .param("page", "2")
                .param("size", "5"))
            .andExpect(status().isOk())
            .andExpect(jsonPath("$.pagination.page").value(2))
            .andExpect(jsonPath("$.pagination.size").value(5));
    }

    // ================================
    // GET /api/v2/posts/{id}
    // ================================

    @Test
    @DisplayName("GET /api/v2/posts/{id} - should return post when found")
    void getPost_shouldReturnPost_whenFound() throws Exception {
        given(postService.findById(1L)).willReturn(Optional.of(testPost));

        mockMvc.perform(get("/api/v2/posts/1"))
            .andExpect(status().isOk())
            .andExpect(jsonPath("$.id").value(1))
            .andExpect(jsonPath("$.title").value("Test Post"))
            .andExpect(jsonPath("$.slug").value("test-post"))
            .andExpect(jsonPath("$.author.username").value("testuser"));
    }

    @Test
    @DisplayName("GET /api/v2/posts/{id} - should return 404 when not found")
    void getPost_shouldReturn404_whenNotFound() throws Exception {
        given(postService.findById(99L)).willReturn(Optional.empty());

        mockMvc.perform(get("/api/v2/posts/99"))
            .andExpect(status().isNotFound());
    }

    // ================================
    // POST /api/v2/posts
    // ================================

    @Test
    @WithMockUser(username = "testuser", roles = {"AUTHOR"})  // Mock authenticated user
    @DisplayName("POST /api/v2/posts - should create post for authenticated author")
    void createPost_shouldCreatePost_forAuthenticatedAuthor() throws Exception {
        CreatePostRequest request = CreatePostRequest.builder()
            .title("New Post Title")
            .content("New post content here")
            .excerpt("Short excerpt")
            .build();

        given(postService.createPost(any(CreatePostRequest.class), eq("testuser")))
            .willReturn(testPost);

        mockMvc.perform(post("/api/v2/posts")
                .with(csrf())  // Include CSRF token
                .contentType(MediaType.APPLICATION_JSON)
                .content(objectMapper.writeValueAsString(request)))
            .andExpect(status().isCreated())
            .andExpect(header().exists("Location"))
            .andExpect(jsonPath("$.title").value("Test Post"));
    }

    @Test
    @DisplayName("POST /api/v2/posts - should return 401 when not authenticated")
    void createPost_shouldReturn401_whenNotAuthenticated() throws Exception {
        CreatePostRequest request = CreatePostRequest.builder()
            .title("New Post")
            .content("Content")
            .build();

        mockMvc.perform(post("/api/v2/posts")
                .with(csrf())
                .contentType(MediaType.APPLICATION_JSON)
                .content(objectMapper.writeValueAsString(request)))
            .andExpect(status().isUnauthorized());
    }

    @Test
    @WithMockUser(username = "user", roles = {"USER"})
    @DisplayName("POST /api/v2/posts - should return 403 for USER role")
    void createPost_shouldReturn403_forUserRole() throws Exception {
        CreatePostRequest request = CreatePostRequest.builder()
            .title("New Post")
            .content("Content")
            .build();

        mockMvc.perform(post("/api/v2/posts")
                .with(csrf())
                .contentType(MediaType.APPLICATION_JSON)
                .content(objectMapper.writeValueAsString(request)))
            .andExpect(status().isForbidden());
    }

    @Test
    @WithMockUser(username = "author", roles = {"AUTHOR"})
    @DisplayName("POST /api/v2/posts - should return 400 for missing title")
    void createPost_shouldReturn400_forMissingTitle() throws Exception {
        CreatePostRequest request = CreatePostRequest.builder()
            .content("Content only, no title")
            .build();

        mockMvc.perform(post("/api/v2/posts")
                .with(csrf())
                .contentType(MediaType.APPLICATION_JSON)
                .content(objectMapper.writeValueAsString(request)))
            .andExpect(status().isBadRequest());
    }

    // ================================
    // DELETE /api/v2/posts/{id}
    // ================================

    @Test
    @WithMockUser(username = "admin", roles = {"ADMIN"})
    @DisplayName("DELETE /api/v2/posts/{id} - should delete post for admin")
    void deletePost_shouldDeletePost_forAdmin() throws Exception {
        willDoNothing().given(postService).deletePost(eq(1L), eq("admin"));

        mockMvc.perform(delete("/api/v2/posts/1")
                .with(csrf()))
            .andExpect(status().isNoContent());

        then(postService).should(times(1)).deletePost(1L, "admin");
    }
}
```

---

## 5. @DataJpaTest - Repository Layer Tests

```java
package com.example.blog.repository;

import com.example.blog.entity.*;
import org.junit.jupiter.api.*;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.test.autoconfigure.orm.jpa.*;
import org.springframework.data.domain.Page;
import org.springframework.data.domain.PageRequest;
import org.springframework.test.context.TestPropertySource;

import java.time.LocalDateTime;
import java.util.List;
import java.util.Optional;
import java.util.Set;

import static org.assertj.core.api.Assertions.*;

// @DataJpaTest:
// - Loads only JPA components (repositories, entities, JPA config)
// - Uses in-memory H2 by default
// - Wraps each test in transaction (auto-rollback after each test)
// - Does NOT load full Spring context
@DataJpaTest
@AutoConfigureTestDatabase(replace = AutoConfigureTestDatabase.Replace.NONE)  // Use actual DB config
// OR for H2 (default):
// @DataJpaTest  // Just this - uses H2
@DisplayName("PostRepository Tests")
class PostRepositoryTest {

    @Autowired
    private PostRepository postRepository;

    @Autowired
    private UserRepository userRepository;

    @Autowired
    private CategoryRepository categoryRepository;

    @Autowired
    private TagRepository tagRepository;

    @Autowired
    private TestEntityManager entityManager;  // For setup/verification without repository

    private User savedUser;
    private Category savedCategory;
    private Tag savedTag;

    @BeforeEach
    void setUp() {
        savedUser = userRepository.save(User.builder()
            .username("testuser")
            .email("test@example.com")
            .password("hashedpw")
            .build());

        savedCategory = categoryRepository.save(Category.builder()
            .name("Technology")
            .slug("technology")
            .build());

        savedTag = tagRepository.save(Tag.builder()
            .name("spring")
            .slug("spring")
            .build());
    }

    private Post createAndSavePost(String title, PostStatus status) {
        Post post = Post.builder()
            .title(title)
            .slug(title.toLowerCase().replace(" ", "-"))
            .content("Content for " + title)
            .status(status)
            .author(savedUser)
            .category(savedCategory)
            .tags(Set.of(savedTag))
            .publishedAt(status == PostStatus.PUBLISHED ? LocalDateTime.now() : null)
            .build();
        return postRepository.save(post);
    }

    @Test
    @DisplayName("save should persist post")
    void save_shouldPersistPost() {
        Post post = createAndSavePost("My Test Post", PostStatus.DRAFT);

        assertThat(post.getId()).isNotNull();
        assertThat(post.getTitle()).isEqualTo("My Test Post");
        assertThat(post.getStatus()).isEqualTo(PostStatus.DRAFT);
    }

    @Test
    @DisplayName("findBySlug should return post for existing slug")
    void findBySlug_shouldReturnPost_forExistingSlug() {
        createAndSavePost("Spring Boot Guide", PostStatus.PUBLISHED);
        entityManager.flush();  // Ensure data is persisted to DB

        Optional<Post> result = postRepository.findBySlug("spring-boot-guide");

        assertThat(result).isPresent();
        assertThat(result.get().getTitle()).isEqualTo("Spring Boot Guide");
    }

    @Test
    @DisplayName("findBySlug should return empty for non-existent slug")
    void findBySlug_shouldReturnEmpty_forNonExistentSlug() {
        Optional<Post> result = postRepository.findBySlug("non-existent-slug");
        assertThat(result).isEmpty();
    }

    @Test
    @DisplayName("findByStatus should return only posts with given status")
    void findByStatus_shouldReturnFilteredPosts() {
        createAndSavePost("Published Post 1", PostStatus.PUBLISHED);
        createAndSavePost("Published Post 2", PostStatus.PUBLISHED);
        createAndSavePost("Draft Post", PostStatus.DRAFT);
        entityManager.flush();
        entityManager.clear();  // Clear first-level cache

        Page<Post> result = postRepository.findByStatus(PostStatus.PUBLISHED, PageRequest.of(0, 10));

        assertThat(result.getContent()).hasSize(2);
        assertThat(result.getContent()).allMatch(p -> p.getStatus() == PostStatus.PUBLISHED);
    }

    @Test
    @DisplayName("existsBySlug should return true for existing slug")
    void existsBySlug_shouldReturnTrue_forExistingSlug() {
        createAndSavePost("Existing Post", PostStatus.DRAFT);
        entityManager.flush();

        boolean exists = postRepository.existsBySlug("existing-post");

        assertThat(exists).isTrue();
    }

    @Test
    @DisplayName("findByAuthorUsername should return posts for given author")
    void findByAuthorUsername_shouldReturnAuthorPosts() {
        // Create second user
        User anotherUser = userRepository.save(User.builder()
            .username("another")
            .email("another@example.com")
            .password("pw")
            .build());

        Post post1 = createAndSavePost("First Post", PostStatus.PUBLISHED);
        Post post2 = createAndSavePost("Second Post", PostStatus.PUBLISHED);

        Post otherPost = postRepository.save(Post.builder()
            .title("Other User Post")
            .slug("other-user-post")
            .content("Content")
            .status(PostStatus.PUBLISHED)
            .author(anotherUser)
            .build());

        entityManager.flush();
        entityManager.clear();

        Page<Post> result = postRepository.findByAuthorUsername("testuser", PageRequest.of(0, 10));

        assertThat(result.getContent()).hasSize(2);
        assertThat(result.getContent())
            .allMatch(p -> p.getAuthor().getUsername().equals("testuser"));
    }

    @Test
    @DisplayName("incrementViewCount should increment the count by 1")
    void incrementViewCount_shouldIncrementCount() {
        Post post = createAndSavePost("Popular Post", PostStatus.PUBLISHED);
        Long postId = post.getId();
        entityManager.flush();
        entityManager.clear();

        postRepository.incrementViewCount(postId);
        entityManager.flush();
        entityManager.clear();

        Post updated = postRepository.findById(postId).orElseThrow();
        assertThat(updated.getViewCount()).isEqualTo(1);
    }

    @Test
    @DisplayName("findByTagSlugAndStatus should return posts with given tag")
    void findByTagSlugAndStatus_shouldReturnTaggedPosts() {
        Tag anotherTag = tagRepository.save(Tag.builder()
            .name("java")
            .slug("java")
            .build());

        Post taggedPost = createAndSavePost("Spring Post", PostStatus.PUBLISHED);
        // taggedPost already has 'spring' tag from createAndSavePost

        Post javaPost = postRepository.save(Post.builder()
            .title("Java Post")
            .slug("java-post")
            .content("Java content")
            .status(PostStatus.PUBLISHED)
            .author(savedUser)
            .tags(Set.of(anotherTag))
            .publishedAt(LocalDateTime.now())
            .build());

        entityManager.flush();
        entityManager.clear();

        Page<Post> result = postRepository.findByTagSlugAndStatus(
            "spring", PostStatus.PUBLISHED, PageRequest.of(0, 10));

        assertThat(result.getContent()).hasSize(1);
        assertThat(result.getContent().get(0).getSlug()).isEqualTo("spring-post");
    }

    @Test
    @DisplayName("delete should remove post")
    void delete_shouldRemovePost() {
        Post post = createAndSavePost("To Delete", PostStatus.DRAFT);
        Long postId = post.getId();
        entityManager.flush();

        postRepository.deleteById(postId);
        entityManager.flush();

        assertThat(postRepository.findById(postId)).isEmpty();
    }
}
```

---

## 6. @SpringBootTest - Integration Tests

```java
package com.example.blog;

import com.example.blog.dto.*;
import com.fasterxml.jackson.databind.ObjectMapper;
import org.junit.jupiter.api.*;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.test.autoconfigure.web.servlet.AutoConfigureMockMvc;
import org.springframework.boot.test.context.SpringBootTest;
import org.springframework.http.MediaType;
import org.springframework.test.annotation.DirtiesContext;
import org.springframework.test.context.ActiveProfiles;
import org.springframework.test.web.servlet.MockMvc;
import org.springframework.test.web.servlet.MvcResult;
import org.springframework.transaction.annotation.Transactional;

import static org.assertj.core.api.Assertions.*;
import static org.springframework.test.web.servlet.request.MockMvcRequestBuilders.*;
import static org.springframework.test.web.servlet.result.MockMvcResultHandlers.print;
import static org.springframework.test.web.servlet.result.MockMvcResultMatchers.*;

// @SpringBootTest loads FULL application context
// Much slower than @WebMvcTest but tests real integration
@SpringBootTest(webEnvironment = SpringBootTest.WebEnvironment.RANDOM_PORT)
@AutoConfigureMockMvc
@ActiveProfiles("test")  // Use application-test.properties
@TestMethodOrder(MethodOrderer.OrderAnnotation.class)
@DisplayName("Blog API Integration Tests")
class BlogIntegrationTest {

    @Autowired
    private MockMvc mockMvc;

    @Autowired
    private ObjectMapper objectMapper;

    @Autowired
    private UserRepository userRepository;

    @Autowired
    private PostRepository postRepository;

    // Store tokens between tests
    private static String accessToken;
    private static Long createdPostId;

    @Test
    @Order(1)
    @DisplayName("Register new user")
    void testRegister() throws Exception {
        RegisterRequest request = new RegisterRequest("integrationuser", "integration@test.com", "password123");

        mockMvc.perform(post("/api/auth/register")
                .contentType(MediaType.APPLICATION_JSON)
                .content(objectMapper.writeValueAsString(request)))
            .andExpect(status().isCreated())
            .andExpect(jsonPath("$.username").value("integrationuser"))
            .andExpect(jsonPath("$.email").value("integration@test.com"));

        assertThat(userRepository.findByUsername("integrationuser")).isPresent();
    }

    @Test
    @Order(2)
    @DisplayName("Login and receive JWT token")
    void testLogin() throws Exception {
        LoginRequest request = new LoginRequest("integrationuser", "password123");

        MvcResult result = mockMvc.perform(post("/api/auth/login")
                .contentType(MediaType.APPLICATION_JSON)
                .content(objectMapper.writeValueAsString(request)))
            .andExpect(status().isOk())
            .andExpect(jsonPath("$.accessToken").exists())
            .andExpect(jsonPath("$.refreshToken").exists())
            .andExpect(jsonPath("$.tokenType").value("Bearer"))
            .andReturn();

        String responseBody = result.getResponse().getContentAsString();
        AuthResponse authResponse = objectMapper.readValue(responseBody, AuthResponse.class);
        accessToken = authResponse.getAccessToken();

        assertThat(accessToken).isNotBlank();
    }

    @Test
    @Order(3)
    @DisplayName("Get current user with JWT")
    void testGetCurrentUser() throws Exception {
        mockMvc.perform(get("/api/auth/me")
                .header("Authorization", "Bearer " + accessToken))
            .andExpect(status().isOk())
            .andExpect(jsonPath("$.username").value("integrationuser"));
    }

    @Test
    @Order(4)
    @DisplayName("Access protected endpoint without token returns 401")
    void testProtectedEndpointWithoutToken() throws Exception {
        mockMvc.perform(get("/api/auth/me"))
            .andExpect(status().isUnauthorized());
    }

    @Test
    @Order(5)
    @DisplayName("Get public posts without authentication")
    void testGetPublicPosts() throws Exception {
        mockMvc.perform(get("/api/v2/posts"))
            .andExpect(status().isOk())
            .andExpect(jsonPath("$.data").isArray());
    }
}
```

---

## 7. TestContainers

```java
package com.example.blog;

import org.junit.jupiter.api.*;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.test.autoconfigure.jdbc.AutoConfigureTestDatabase;
import org.springframework.boot.test.autoconfigure.orm.jpa.DataJpaTest;
import org.springframework.test.context.DynamicPropertyRegistry;
import org.springframework.test.context.DynamicPropertySource;
import org.testcontainers.containers.PostgreSQLContainer;
import org.testcontainers.junit.jupiter.Container;
import org.testcontainers.junit.jupiter.Testcontainers;

import static org.assertj.core.api.Assertions.*;

// TestContainers: spin up a real Docker container for testing
@DataJpaTest
@Testcontainers
@AutoConfigureTestDatabase(replace = AutoConfigureTestDatabase.Replace.NONE)
@DisplayName("PostRepository with Real PostgreSQL")
class PostRepositoryPostgresTest {

    // Spin up PostgreSQL container (shared across all tests in this class)
    @Container
    static PostgreSQLContainer<?> postgres = new PostgreSQLContainer<>("postgres:16-alpine")
        .withDatabaseName("testdb")
        .withUsername("testuser")
        .withPassword("testpass")
        .withInitScript("schema.sql");  // Optional: run SQL on startup

    // Register container properties dynamically
    @DynamicPropertySource
    static void configureProperties(DynamicPropertyRegistry registry) {
        registry.add("spring.datasource.url", postgres::getJdbcUrl);
        registry.add("spring.datasource.username", postgres::getUsername);
        registry.add("spring.datasource.password", postgres::getPassword);
    }

    @Autowired
    private PostRepository postRepository;

    @Autowired
    private UserRepository userRepository;

    @Test
    @DisplayName("Should work with real PostgreSQL full-text search")
    void testFullTextSearch() {
        // This test uses actual PostgreSQL, not H2
        // Can test PostgreSQL-specific features
        User user = userRepository.save(User.builder()
            .username("author")
            .email("author@test.com")
            .password("pw")
            .build());

        postRepository.save(Post.builder()
            .title("Spring Boot Testing Guide")
            .slug("spring-boot-testing")
            .content("Learn how to test Spring Boot applications effectively")
            .status(PostStatus.PUBLISHED)
            .author(user)
            .build());

        // Test can use PostgreSQL-specific queries
        List<Post> results = postRepository.searchPublished("testing");
        assertThat(results).isNotEmpty();
        assertThat(results.get(0).getTitle()).contains("Testing");
    }
}

// Full integration test with TestContainers
@SpringBootTest(webEnvironment = SpringBootTest.WebEnvironment.RANDOM_PORT)
@AutoConfigureMockMvc
@Testcontainers
@ActiveProfiles("integration")
class FullIntegrationTest {

    @Container
    static PostgreSQLContainer<?> postgres = new PostgreSQLContainer<>("postgres:16-alpine")
        .withDatabaseName("blogdb")
        .withUsername("admin")
        .withPassword("secret");

    @DynamicPropertySource
    static void registerPgProperties(DynamicPropertyRegistry registry) {
        registry.add("spring.datasource.url", postgres::getJdbcUrl);
        registry.add("spring.datasource.username", postgres::getUsername);
        registry.add("spring.datasource.password", postgres::getPassword);
        registry.add("spring.jpa.hibernate.ddl-auto", () -> "create-drop");
    }

    @Autowired
    private MockMvc mockMvc;

    @Test
    void testCompleteWorkflow() throws Exception {
        // Full workflow test with real database
        // ...
    }
}
```

---

## 8. Testing Security

```java
@WebMvcTest(PostController.class)
class PostSecurityTest {

    @Autowired
    private MockMvc mockMvc;

    @MockBean
    private PostService postService;

    @MockBean
    private JwtTokenService jwtTokenService;

    @MockBean
    private CustomUserDetailsService userDetailsService;

    // Using @WithMockUser - sets SecurityContext with mock user
    @Test
    @WithMockUser(username = "user", roles = {"USER"})
    void accessWithUserRole_shouldDenyPostCreation() throws Exception {
        mockMvc.perform(post("/api/v2/posts")
                .with(csrf())
                .contentType(MediaType.APPLICATION_JSON)
                .content("{\"title\":\"T\",\"content\":\"C\"}"))
            .andExpect(status().isForbidden());
    }

    @Test
    @WithMockUser(username = "author", roles = {"AUTHOR"})
    void accessWithAuthorRole_shouldAllowPostCreation() throws Exception {
        Post post = Post.builder().id(1L).title("T").author(
            User.builder().id(1L).username("author").build()
        ).build();
        given(postService.createPost(any(), eq("author"))).willReturn(post);

        mockMvc.perform(post("/api/v2/posts")
                .with(csrf())
                .contentType(MediaType.APPLICATION_JSON)
                .content("{\"title\":\"Test Title\",\"content\":\"Content here\"}"))
            .andExpect(status().isCreated());
    }

    @Test
    @WithMockUser(username = "admin", roles = {"ADMIN"})
    void adminShouldDeletePost() throws Exception {
        willDoNothing().given(postService).deletePost(eq(1L), eq("admin"));

        mockMvc.perform(delete("/api/v2/posts/1").with(csrf()))
            .andExpect(status().isNoContent());
    }

    // Using @WithUserDetails - loads actual UserDetails from UserDetailsService
    @Test
    @WithUserDetails(value = "testuser", userDetailsServiceBeanName = "customUserDetailsService")
    void withUserDetails_shouldLoadActualUserDetails() throws Exception {
        // This uses your actual UserDetailsService to load the user
        mockMvc.perform(get("/api/auth/me"))
            .andExpect(status().isOk());
    }

    // Testing with JWT token
    @Test
    void withJwtToken_shouldAuthenticate() throws Exception {
        // Mock the JWT validation
        String token = "valid.jwt.token";
        given(jwtTokenService.validateTokenSignature(token)).willReturn(true);
        given(jwtTokenService.isAccessToken(token)).willReturn(true);
        given(jwtTokenService.extractUsername(token)).willReturn("jwtuser");
        given(jwtTokenService.isTokenExpired(token)).willReturn(false);

        UserDetails userDetails = org.springframework.security.core.userdetails.User
            .withUsername("jwtuser")
            .password("pw")
            .roles("USER")
            .build();
        given(userDetailsService.loadUserByUsername("jwtuser")).willReturn(userDetails);
        given(jwtTokenService.isTokenValid(eq(token), any())).willReturn(true);

        mockMvc.perform(get("/api/auth/me")
                .header("Authorization", "Bearer " + token))
            .andExpect(status().isOk());
    }
}
```

---

## 9. @MockBean vs @SpyBean

```java
@SpringBootTest
@AutoConfigureMockMvc
class MockBeanVsSpyBeanTest {

    @Autowired
    private MockMvc mockMvc;

    // @MockBean: creates a pure Mockito mock, replaces the real bean in context
    // All methods return null/0/false by default unless you stub them
    @MockBean
    private EmailService emailService;

    // @SpyBean: wraps the real bean, calls real methods unless overridden
    // Useful when you want to verify interactions but use real implementation
    @SpyBean
    private PostService postService;

    @Test
    void mockBean_shouldNotCallRealMethod() {
        // emailService is mocked - no real emails sent
        // we can verify it was called
        willDoNothing().given(emailService).sendWelcomeEmail(any());

        authService.register(new RegisterRequest("user", "test@test.com", "pw"));

        then(emailService).should(times(1)).sendWelcomeEmail("test@test.com");
    }

    @Test
    void spyBean_callsRealMethodByDefault() {
        // postService.findById calls REAL implementation
        // but we can still verify the call
        Optional<Post> post = postService.findById(1L);

        // Verify the method was called
        then(postService).should(times(1)).findById(1L);
    }

    @Test
    void spyBean_canOverrideSpecificMethod() {
        // Override just one method of the real service
        doReturn(List.of()).when(postService).findByStatus(PostStatus.DRAFT);

        // Other methods still use real implementation
        List<Post> result = postService.findByStatus(PostStatus.DRAFT);
        assertThat(result).isEmpty();
    }
}
```

---

## 10. RestAssured

```java
@SpringBootTest(webEnvironment = SpringBootTest.WebEnvironment.RANDOM_PORT)
@ActiveProfiles("test")
class PostApiRestAssuredTest {

    @LocalServerPort
    private int port;

    @Autowired
    private ObjectMapper objectMapper;

    private RequestSpecification requestSpec;
    private String authToken;

    @BeforeEach
    void setUp() {
        RestAssured.port = port;
        RestAssured.basePath = "/api/v2";

        requestSpec = new RequestSpecBuilder()
            .setContentType(ContentType.JSON)
            .setAccept(ContentType.JSON)
            .build();

        // Get JWT token
        authToken = given(requestSpec)
            .basePath("/api/auth")
            .body(new LoginRequest("admin", "admin123"))
            .when()
            .post("/login")
            .then()
            .statusCode(200)
            .extract()
            .path("accessToken");
    }

    @Test
    @DisplayName("GET /posts - should return posts with correct structure")
    void getPostsShouldReturnCorrectStructure() {
        given(requestSpec)
            .when()
            .get("/posts")
            .then()
            .statusCode(200)
            .body("data", notNullValue())
            .body("pagination.page", equalTo(0))
            .body("pagination.size", equalTo(10))
            .body("pagination.totalElements", greaterThanOrEqualTo(0));
    }

    @Test
    @DisplayName("POST /posts - should create post and return Location header")
    void createPostShouldReturnLocation() {
        Map<String, Object> requestBody = Map.of(
            "title", "RestAssured Test Post",
            "content", "Content created via RestAssured",
            "excerpt", "Short excerpt"
        );

        String location = given(requestSpec)
            .header("Authorization", "Bearer " + authToken)
            .body(requestBody)
            .when()
            .post("/posts")
            .then()
            .statusCode(201)
            .header("Location", notNullValue())
            .body("title", equalTo("RestAssured Test Post"))
            .body("id", notNullValue())
            .extract()
            .header("Location");

        assertThat(location).contains("/api/v2/posts/");

        // Follow the Location header to get the created post
        given(requestSpec)
            .when()
            .get(location)
            .then()
            .statusCode(200)
            .body("title", equalTo("RestAssured Test Post"));
    }

    @Test
    @DisplayName("DELETE /posts/{id} - should delete post")
    void deletePostShouldSucceed() {
        // First create a post
        Map<String, Object> createBody = Map.of(
            "title", "To Delete",
            "content", "Will be deleted"
        );

        Integer postId = given(requestSpec)
            .header("Authorization", "Bearer " + authToken)
            .body(createBody)
            .when()
            .post("/posts")
            .then()
            .statusCode(201)
            .extract()
            .path("id");

        // Then delete it
        given(requestSpec)
            .header("Authorization", "Bearer " + authToken)
            .when()
            .delete("/posts/" + postId)
            .then()
            .statusCode(204);

        // Verify it's gone
        given(requestSpec)
            .when()
            .get("/posts/" + postId)
            .then()
            .statusCode(404);
    }
}
```

---

## 11. Test Data Builders

```java
// Builder pattern for test data
public class PostTestBuilder {

    private Long id = 1L;
    private String title = "Default Test Title";
    private String slug = "default-test-title";
    private String content = "Default test content";
    private String excerpt = "Default excerpt";
    private PostStatus status = PostStatus.PUBLISHED;
    private int viewCount = 0;
    private boolean featured = false;
    private LocalDateTime publishedAt = LocalDateTime.now().minusDays(1);
    private User author;
    private Category category;
    private Set<Tag> tags = new HashSet<>();

    public static PostTestBuilder aPost() {
        return new PostTestBuilder();
    }

    public PostTestBuilder withId(Long id) {
        this.id = id;
        return this;
    }

    public PostTestBuilder withTitle(String title) {
        this.title = title;
        this.slug = title.toLowerCase().replace(" ", "-");
        return this;
    }

    public PostTestBuilder withStatus(PostStatus status) {
        this.status = status;
        if (status != PostStatus.PUBLISHED) {
            this.publishedAt = null;
        }
        return this;
    }

    public PostTestBuilder withAuthor(User author) {
        this.author = author;
        return this;
    }

    public PostTestBuilder withCategory(Category category) {
        this.category = category;
        return this;
    }

    public PostTestBuilder withTags(Tag... tags) {
        this.tags = Set.of(tags);
        return this;
    }

    public PostTestBuilder asDraft() {
        return withStatus(PostStatus.DRAFT);
    }

    public PostTestBuilder asPublished() {
        return withStatus(PostStatus.PUBLISHED);
    }

    public PostTestBuilder asFeatured() {
        this.featured = true;
        return this;
    }

    public Post build() {
        if (author == null) {
            author = UserTestBuilder.anAuthor().build();
        }
        return Post.builder()
            .id(id)
            .title(title)
            .slug(slug)
            .content(content)
            .excerpt(excerpt)
            .status(status)
            .viewCount(viewCount)
            .featured(featured)
            .publishedAt(publishedAt)
            .author(author)
            .category(category)
            .tags(tags)
            .createdAt(LocalDateTime.now().minusDays(2))
            .updatedAt(LocalDateTime.now().minusHours(1))
            .build();
    }
}

public class UserTestBuilder {

    private Long id = 1L;
    private String username = "testuser";
    private String email = "test@example.com";
    private String password = "hashedpassword";
    private Set<Role> roles = Set.of(Role.USER);

    public static UserTestBuilder aUser() {
        return new UserTestBuilder();
    }

    public static UserTestBuilder anAuthor() {
        return new UserTestBuilder().withRoles(Set.of(Role.AUTHOR));
    }

    public static UserTestBuilder anAdmin() {
        return new UserTestBuilder()
            .withUsername("admin")
            .withEmail("admin@example.com")
            .withRoles(Set.of(Role.ADMIN));
    }

    public UserTestBuilder withId(Long id) { this.id = id; return this; }
    public UserTestBuilder withUsername(String username) { this.username = username; return this; }
    public UserTestBuilder withEmail(String email) { this.email = email; return this; }
    public UserTestBuilder withRoles(Set<Role> roles) { this.roles = roles; return this; }

    public User build() {
        return User.builder()
            .id(id)
            .username(username)
            .email(email)
            .password(password)
            .roles(roles)
            .enabled(true)
            .build();
    }
}

// Using test builders in tests
@Test
void createPost_shouldReturnPost() {
    User author = UserTestBuilder.anAuthor().withUsername("blogauthor").build();

    Post expectedPost = PostTestBuilder.aPost()
        .withTitle("My Spring Boot Guide")
        .withAuthor(author)
        .asPublished()
        .asFeatured()
        .build();

    given(postRepository.save(any())).willReturn(expectedPost);

    // ... test code
}
```

---

## 12. Test Configuration

```properties
# src/test/resources/application-test.properties
spring.datasource.url=jdbc:h2:mem:testdb;DB_CLOSE_DELAY=-1;MODE=PostgreSQL
spring.datasource.driver-class-name=org.h2.Driver
spring.jpa.hibernate.ddl-auto=create-drop
spring.jpa.show-sql=false

# JWT for tests
app.jwt.secret=testSecretKeyThatIsLongEnoughForHmacSha256Algorithm
app.jwt.expiration=3600000
app.jwt.refresh-expiration=86400000

# Disable security for certain tests (or configure test security)
# logging.level.org.springframework.security=DEBUG
```

```java
// Test configuration class
@TestConfiguration
public class TestSecurityConfig {

    @Bean
    @Primary  // Override production bean in tests
    public PasswordEncoder testPasswordEncoder() {
        return new BCryptPasswordEncoder(4);  // Lower cost factor for faster tests
    }
}

// Use in test class
@SpringBootTest
@Import(TestSecurityConfig.class)
class AuthServiceTest {
    // ...
}
```

---

## 13. Complete Test Suite Overview

```java
// Summary of all test types and when to use them

/*
 * 1. @ExtendWith(MockitoExtension.class) - Pure Unit Tests
 *    - Test: Service classes, utility classes, domain logic
 *    - Speed: Very fast (no Spring context)
 *    - When: Testing business logic in isolation
 *
 * 2. @WebMvcTest - Controller Tests
 *    - Test: HTTP mapping, request/response, validation, security rules
 *    - Speed: Fast (partial context)
 *    - When: Testing REST endpoints, input validation
 *
 * 3. @DataJpaTest - Repository Tests
 *    - Test: JPA queries, entity relationships, constraints
 *    - Speed: Medium (JPA context + in-memory DB)
 *    - When: Testing custom queries, derived methods
 *
 * 4. @SpringBootTest - Integration Tests
 *    - Test: Full application flow, multiple layers together
 *    - Speed: Slow (full context)
 *    - When: Testing end-to-end scenarios
 *
 * 5. @SpringBootTest + TestContainers
 *    - Test: Real database behavior, migrations, DB-specific features
 *    - Speed: Slowest (Docker containers)
 *    - When: Production-like database testing
 */

// Test naming convention
// methodName_shouldExpectedBehavior_whenCondition()
// OR
// givenSetup_whenAction_thenExpected()

// Assertion styles
// JUnit 5: Assertions.assertEquals, assertThrows
// AssertJ: assertThat(x).isEqualTo(y)  (recommended - more readable)
// Hamcrest: assertThat(x, equalTo(y))  (used in MockMvc expectations)
```

---

## 14. Summary Table

| Test Type | Annotation | What Loads | Speed | Use For |
|---|---|---|---|---|
| Unit test | `@ExtendWith(MockitoExtension)` | Nothing (pure Java) | Fastest | Business logic |
| Controller test | `@WebMvcTest` | Web layer only | Fast | HTTP mapping, security |
| Repository test | `@DataJpaTest` | JPA layer only | Medium | Queries, constraints |
| Integration test | `@SpringBootTest` | Full context | Slow | End-to-end flows |
| Real DB test | `@SpringBootTest` + TestContainers | Full context + Docker | Slowest | DB-specific features |
| Mock bean | `@MockBean` | Replace with mock | N/A | Isolate dependencies |
| Spy bean | `@SpyBean` | Wrap real bean | N/A | Partial mocking |
| Mock user | `@WithMockUser` | N/A | N/A | Security tests |
| Real user details | `@WithUserDetails` | N/A | N/A | Custom security tests |
| REST client | RestAssured | N/A | N/A | API integration tests |

---

## Course Completion

Congratulations! You have completed the Java Spring Boot course. Here is a summary of all parts covered:

| Part | Topic |
|---|---|
| 001-010 | Java fundamentals (syntax, OOP, exceptions) |
| 011-018 | Advanced Java (generics, streams, concurrency, patterns) |
| 019-020 | Testing and build tools |
| 021-024 | Spring Framework basics (IoC, AOP, MVC, Actuator) |
| 025 | Spring Data JPA (entities, repositories, queries) |
| 026 | Spring Security (authentication, authorization, RBAC) |
| 027 | JWT Authentication (tokens, filters, stateless) |
| 028 | Advanced Spring Data (specs, projections, caching) |
| 029 | Advanced REST API design (versioning, HATEOAS, docs) |
| 030 | Spring Testing (unit, integration, security tests) |
