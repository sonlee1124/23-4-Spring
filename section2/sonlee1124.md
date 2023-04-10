generate커멘드 > alt + insert

1. 프로젝트 생성
1-1) https://start.spring.io/ 에 접속하여 원하는 파일을 생성한다.
1-2) 1-1에서 만든 파일 중 bulid.gradle을 선택하여 파일을 오픈한다.

2. 비즈니스 요구사항과 설계
- 회원
	- 회원 가입, 회원 조회
	- 회원의 등급 : 일반, VIP
	- 회원의 데이터는 자체 DB구축 가능, 외부 시스템과 연동 가능
- 주문과 할인
	- 회원은 상품 가능
	- 회원 등급에 따라 할인 가능
	- 모든 VIP는 1000원 할인(고정 금액 할인) - 추후에 변경 가능
	- 할인은 변경 가능성이 매우 높음. 아직 정책을 정한건 아니라 오픈 직전까지 생각하기
	※최악의 경우 안할수도 있음^^
인터페이스를 만들고 언제든지 갈아 끼울 수 있게 설계하기!

3.회원 도메인 설계
- 회원가입, 회원 조회 가능
- 회원의 등급 : 일반, VIP
- 회원의 데이터는 자체 DB구축 가능, 외부 시스템과 연동 가능

회원 도메인 협력 관계
클라이언트 > 회원서비스(회원가입, 회원조회) > 회원저장소의 역할(메모리 회원 저장소, DB회원저장소, 외부시스템 연동 회원 저장소 中 1개 선택)
- 메모리 회원 저장소일 경우 껐다 키면 메모리 날라는 현상이 나타남

회원 클래스 다이어그램
- MemberService라는 역할을 interface로 만들기
- MemberSeriveImpl(회원 서비스)로 구현체를 만들기
- MemberRepository(회원 저장소)라는 역할 interface만들기
- 구현클래스로 MemoryMemberRepository(메모리 회원 저장소) OR DbMemberRepository(DB회원저장소)를 구현

회원 객체 다이어그램
클라이언트 > 회원 서비스 > 메모리 회원 저장소

4. 회원 도메인 개발
- 회원 등급 
public enum Grade {
    BASIC,
    VIP
}

- 회원 객체
public class Member {
    private Long id;
    private String name;
    private Grade grade;

    public Member(Long id, String name, Grade grade) {
        this.id = id;
        this.name = name;
        this.grade = grade;
    }

    public Long getId() {
        return id;
    }

    public void setId(Long id) {
        this.id = id;
    }

    public String getName() {
        return name;
    }

    public void setName(String name) {
        this.name = name;
    }

    public Grade getGrade() {
        return grade;
    }

    public void setGrade(Grade grade) {
        this.grade = grade;
    }
}

- 회원 저장소 인터페이스
public interface MemberRepository {
    //저장
    void save(Member member);
    //아이디 찾기
    Member findById(Long memberId);
}

- 메모리 멤버 저장소 구현체
public class MemoryMemberRepository implements MemberRepository{

    //회원의 정보를 저장하기 위한 Map생성
    private static Map<Long, Member> store = new HashMap<>();
    
    //저장
    @Override
    public void save(Member member) {
        store.put(member.getId(), member);
    }

    //아이디 찾기
    @Override
    public Member findById(Long memberId) {
        return store.get(memberId);
    }
}

- 회원 서비스 인터페이스
public interface MemberService {
	//회원 가입
    void join(Member member);
	//회원 조회
    Member findMember(Long memberId);
}

- 회원 서비스 구현체
public class MemberServiceImpl implements MemberService{
    //MemoryMemberRepository에 있는 것들이 호출
    private final MemberRepository memberRepository = new MemoryMemberRepository();
    
    //회원 가입 메서드
    @Override
    public void join(Member member) {
        memberRepository.save(member);
    }

    //회원 조회 메서드
    @Override
    public Member findMember(Long memberId) {
        return memberRepository.findById(memberId);
    }
}

5. 회원 도메인 실행과 테스트
psvm > main메서드 생성

1. main메서드로 실행해서 확인하는 방법
public static void main(String[] args) {
        MemberService memberService = new MemberServiceImpl();
        Member member = new Member(1L, "memberA", Grade.VIP);
        memberService.join(member);

        Member fineMember = memberService.findMember(1L);
        System.out.println("member = " + member.getName());
        System.out.println("fineMember = " + fineMember.getName());
    }

2. main메서드 보단 test코드에서 junit 테스트를 활용하여 사용하는게 더 좋은 방법이다
public class MemberServiceTest {

    MemberService memberService = new MemberServiceImpl();
    @Test
    void join(){
        //given ~~한게 주어졌을 때
        Member member = new Member(1L, "memberA", Grade.VIP);

        //when ~~했을 때
        memberService.join(member);
        Member fineMember = memberService.findMember(1L);
        //then ~~게 된다
        //member만든값과 findMember가 맞으면 통과
        Assertions.assertThat(member).isEqualTo(fineMember);
    }
}

6. 주문과 할인 도메인 설꼐
- 주문과 할인
	- 회원은 상품 가능
	- 회원 등급에 따라 할인 가능
	- 모든 VIP는 1000원 할인(고정 금액 할인) - 추후에 변경 가능
	- 할인은 변경 가능성이 매우 높음. 아직 정책을 정한건 아니라 오픈 직전까지 생각하기
	※최악의 경우 안할수도 있음^^
	
클라이언트 > 주문 서비스 역할 > 회원을 조회(회원 저장소) > 등급 확인 후 할인 적용 > 주문 결과를 반환 > 클라이언트
1. 주문 생성 : 클라이언트는 주문 서비스에 주문 생성을 요청
2. 회원 조회 : 할인을 위해 등급을 알아야함. 회원 저장소에서 회원을 조회
3. 할인 적용 : 등급에 따라 할인을 함
4. 주문 결과 반환 : 할인한 결과를 반환

DiscountPolicy 인터페이스 생성 추가 - 할인 
협력 관계를 그대로 재사용 가능

7. 주문과 할인 도메인 개발
-할인 인터페이스
public interface DiscountPolicy {
    //할인 대상 금액
    int discount(Member member, int price);
    
}

- 정액 할인 정책 구현
public class FixDiscountPolicy implements DiscountPolicy{

    private int discountFixAmount = 1000; //1000원 할인
    @Override
    public int discount(Member member, int price) {
        if(member.getGrade()== Grade.VIP){
            return discountFixAmount;
        }else{
            return 0;
        }
    }
}

- 주문 객체
public class Order {

    private Long memberId;
    private String itemName;
    private  int itemPrice;
    private int discountPrice;

    public Order(Long memberId, String itemName, int itemPrice, int discountPrice) {
        this.memberId = memberId;
        this.itemName = itemName;
        this.itemPrice = itemPrice;
        this.discountPrice = discountPrice;
    }
    //계산 로직
    public int calculatePrice(){
        return itemPrice - discountPrice;
    }

    public Long getMemberId() {
        return memberId;
    }

    public void setMemberId(Long memberId) {
        this.memberId = memberId;
    }

    public String getItemName() {
        return itemName;
    }

    public void setItemName(String itemName) {
        this.itemName = itemName;
    }

    public int getItemPrice() {
        return itemPrice;
    }

    public void setItemPrice(int itemPrice) {
        this.itemPrice = itemPrice;
    }

    public int getDiscountPrice() {
        return discountPrice;
    }

    public void setDiscountPrice(int discountPrice) {
        this.discountPrice = discountPrice;
    }

    @Override
    public String toString() {
        return "Order{" +
                "memberId=" + memberId +
                ", itemName='" + itemName + '\'' +
                ", itemPrice=" + itemPrice +
                ", discountPrice=" + discountPrice +
                '}';
    }
}

- 주문 인터페이스
public interface OrderService {
    //주문 생성시 회원아이디, 상품명, 상품가격이 필요
    Order createOrder(Long memberId, String itemName, int itemPrice);
}

- 주문 구현
public class OrderServiceImpl implements OrderService{

    private final MemberRepository memberRepository = new MemoryMemberRepository();
    private final DiscountPolicy discountPolicy = new FixDiscountPolicy();
    @Override
    public Order createOrder(Long memberId, String itemName, int itemPrice) {
        //회원을 조회함
        Member member = memberRepository.findById(memberId);
        //할인 대상 금액
        int discountPrice = discountPolicy.discount(member, itemPrice);
        return new Order(memberId, itemName, itemPrice, discountPrice);
    }
}

8. 주문과 할인 도메인 실행과 테스트
- main메서드로 확인(좋은 방법은 아님!)
public class OrderApp {
    public static void main(String[] args) {
        MemberService memberService = new MemberServiceImpl();
        OrderService orderService = new OrderServiceImpl();

        Long memberId = 1L;
        Member member = new Member(memberId, "memberA", Grade.VIP);
        memberService.join(member);

        Order order = orderService.createOrder(memberId, "itemA", 10000);
        System.out.println("order = " + order);
        System.out.println("order.calculatePrice() = " + order.calculatePrice());
    }
}
-junit으로 테스트
public class OrderServiceTest {

    MemberService memberService = new MemberServiceImpl();
    OrderService orderService = new OrderServiceImpl();

    @Test
    void createOrder(){
        Long memberId = 1L;
        Member member = new Member(memberId, "memberA", Grade.VIP);
        memberService.join(member);
        Order order = orderService.createOrder(memberId, "itemA", 10000);
        Assertions.assertThat(order.getDiscountPrice()).isEqualTo(1000);
    }
}